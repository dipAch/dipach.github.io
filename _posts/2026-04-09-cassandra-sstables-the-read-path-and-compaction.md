---
layout: post
title: "into the eye - cassandra sstables - the read path and compaction stuff"
tags: [cassandra, storage-engine, sstables, compaction, internals]
---

This is **Part 2** of our hands-on SSTable series. [<u>Part 1</u>](/cassandra-sstables-whats-really-on-disk/) covered the file layout, row encoding, and `SerializationHeader`.
Here we watch a read touch up to five components before returning a single row, and then we compact two SSTables by hand to see exactly what LWW (`Last Write Wins`), tombstones, and `gc_grace_seconds` actually do.

## The read path
---

A point lookup in Cassandra doesn't go straight to `Data.db`. It runs through five layers of decision making in order, each one serving as an early escape hatch. The whole sequence lives in `getRowIndexEntry()` inside `<cassandra-package>/io/sstable/format/big/BigTableReader.java`:

```
| Guard   | Stage                 | Check                                   | Result                                 |
|---------|-----------------------|-----------------------------------------|----------------------------------------|
|  1      | Min/Max bounds        | key outside SSTable's token range?      | skip, no I/O                           |
|  2      | Bloom filter          | key definitely absent?                  | skip, no I/O                           |
|  3      | Key cache             | position cached from last lookup?       | jump straight to Data.db               |
|  4      | IndexSummary          | binary search (in-memory)               | nearest sampled position in Index.db   |
|  5      | Index.db scan         | linear scan from sampled position       | RowIndexEntry (byte offset in Data.db) |
```

A few details worth internalizing:

<u><b>Check 1</b></u> uses `getFirst()` and `getLast()` from `Statistics.db`. No file I/O at all. It's why `sstablemetadata` shows `First token` / `Last token`. If your key is outside that range, the SSTable is rejected instantly.

<u><b>Check 2</b></u> (`Filter.db`) is loaded fully into memory when the SSTable is opened. The false positive rate is what `bloom_filter_fp_chance` controls. A false positive lets a missing key fall through to `Index.db`. That's the only cost, but it adds up under read-heavy workloads.

<u><b>Check 3</b></u> is the key cache, an `OHCache` instance (off-heap, outside the JVM heap). It lives outside JVM heap tracking and is accounted for separately. A cache hit means zero `Index.db` I/O: we jump directly to the byte offset in `Data.db`.

<u><b>Check 4</b></u> (`IndexSummary`) is a downsampled copy of `Index.db` keys held in memory, one entry every 128 keys by default. `Binary search` gives you the nearest sampled position to start scanning from. You'll never have to scan more than 128 entries in `Index.db`.

<u><b>Check 5</b></u> is actual disk I/O. The scan reads `(key, RowIndexEntry)` pairs sequentially from the sampled position until it finds your key. `RowIndexEntry` contains the byte offset in `Data.db`, and once found, it's written into the key cache so that <i>Gate 3</i> can fire next time.

> `RowIndexEntry` also stores `IndexInfo` blocks for wide partitions (millions of clustering rows). For small partitions like ours, `serializedSize = 0` - no `IndexInfo`, straight to `Data.db`.

`Index.db` stores one `RowIndexEntry` per partition. For a small partition that entry is just a byte offset into `Data.db` - basically a marker saying start reading from here. That works fine when a partition is just a few hundred bytes.

> For a wide partition, let's say, a time-series table with millions of clustering rows, the partition could be hundreds of megabytes. Seeking to the start and scanning forward to find a clustering row would be a full excruciating linear scan. That's what `IndexInfo` blocks solve.
>
> So the `RowIndexEntry` for a wide partition becomes a list of these blocks: a mini-index within the partition. When reading a specific clustering row, Cassandra binary searches the `IndexInfo` list by clustering range, finds the right block, and seeks directly to its offset.

Still with me?

## Seeing it live with TRACING ON
---

Turn tracing on and query one of the keys from Part 1:

```sql
TRACING ON;

SELECT * FROM lab.events WHERE id = c3e6ef8e-c884-4a7a-9b77-aeddfef8da60;
```

In the trace output look for:

```
Partition index found for sstable 1, size = 0 [ReadStage-3]
```

That's <i>Check 5</i> firing. `size = 0` confirms small partition, no `IndexInfo` blocks.

Now run the **exact same query again**:

```sql
SELECT * FROM lab.events WHERE id = c3e6ef8e-c884-4a7a-9b77-aeddfef8da60;
```

This time you should see:

```
Key cache hit for sstable 1, size = 0 [ReadStage-2]
```

<i>Check 3</i> fired instead this time around. <i>Check 5</i> never ran, no `Index.db` read at all. That's the key cache preventing further trickle down beahavior.

Now query a key that doesn't exist:

```sql
SELECT * FROM lab.events WHERE id = 00000000-0000-0000-0000-000000000000;
```

The trace will show the SSTable was skipped with something like `SSTable 1 is not tracking ... allows skipping`. That's <i>Check 2</i>, the Bloom filter said "definitely not here", so `Index.db` and `Data.db` were never touched.

So far so good.

## What compaction actually does
---

> ## Firstly why even compact? Isn't less work better?
> Every flush produces a new SSTable. Do enough writes and you end up with dozens of them on disk. Each one has its own bloom filter, its own index summary loaded in memory, its own copy of data that may have been overwritten in a newer flush.

A read has to consult all of them. Space and memory overhead grows with every file.

> Compaction is the process that merges SSTables together. It takes N input files, merge-sorts them by token order, resolves conflicts between versions of the same row, purges tombstones that have aged out, and produces a smaller set of output files.
>
> The result is fewer files to check on reads, less memory pressure from index summaries, and reclaimed disk space from dead data.

Let's see compaction do its magic. Insert a second batch and flush to create a second SSTable:

```sql
USE lab;

INSERT INTO events (id, ts, data) VALUES (uuid(), toTimestamp(now()), 'fourth');
INSERT INTO events (id, ts, data) VALUES (uuid(), toTimestamp(now()), 'fifth');

-- Overwrite a row from Part 1 using the exact same clustering value
INSERT INTO events (id, ts, data)
VALUES (c3e6ef8e-c884-4a7a-9b77-aeddfef8da60, '2026-03-14 14:49:58.607Z', 'first-updated');
```

```bash
bin/nodetool flush lab events
```

You now have two `Data.db` files. Dump both - you'll see `c3e6ef8e...` appear in each with a different `liveness_info.tstamp` and a different value:

```
SSTable 1: clustering "2026-03-14 14:49:58.607Z", tstamp "2026-03-14T14:49:58.599939Z", value "first"
SSTable 2: clustering "2026-03-14 14:49:58.607Z", tstamp "2026-03-24T05:36:05.735154Z", value "first-updated"
```

Same partition key, same clustering key: a genuine conflict. Now let's compact:

```bash
bin/nodetool compact lab events
tools/bin/sstabledump \
  ./data/data/lab/events-1b255f4def2540a60000000000000005/pa-*-Data.db
```

The partition for `c3e6ef8e...` now shows only one row - `"first-updated"` with the later timestamp:

```
{
  "partition" : { "key" : [ "c3e6ef8e-c884-4a7a-9b77-aeddfef8da60" ], "position" : 19 },
  "rows" : [
    {
      "clustering" : [ "2026-03-14 14:49:58.607Z" ],
      "liveness_info" : { "tstamp" : "2026-03-24T05:36:05.735154Z" },
      "cells" : [ { "name" : "data", "value" : "first-updated" } ]
    }
  ]
}
```

This is **Last Write Wins**. No locks, no coordination. The logic lives in `reconcile()` inside `<cassandra-package>/db/rows/Cells.java`: one timestamp comparison, higher wins, lower is dropped.

#### <u>How the merge actually works?</u>

Compaction starts one `ISSTableScanner` per input SSTable file, each one a cursor that steps through its SSTable in token order. `ISSTableScanner` extends `UnfilteredPartitionIterator`, so it is an iterator. `<cassandra-package>/io/sstable/format/big/BigTableScanner.java` is the concrete implementation.

Each `ISSTableScanner` is handed as a list to `UnfilteredPartitionIterators.merge()` in `<cassandra-package>/db/partitions/UnfilteredPartitionIterators.java`, which wraps them in a min-heap ordered by partition key. On each step, the heap pops the smallest token. 

When compaction feeds N of these into the merge heap, each one independently advances through its own SSTable at its own pace. The heap coordinates which one yields the next partition.

If two or more scanners land on the same partition key, their row iterators are collected and passed to `UnfilteredRowIterators.merge()`: the same pattern, now ordered by clustering key.

So the merge is two-level: partitions first, rows within a partition second. `Cells.reconcile()` in `<cassandra-package>/db/rows/Cells.java` is only reached when two SSTables have a row at the exact same (partition key, clustering key) address, which is the only moment a conflict exists and `LWW` needs to pick a winner.

> **A quick gotcha worth remembering.**
>
> If instead of reusing the exact same `ts` value, you had inserted with **toTimestamp(now())** - basically a fresh timestamp - you'd get *two separate rows* in the same partition, i.e., same `id`, different `ts` clustering. That would not be an overwrite/mutation, but a new row. Compaction would keep both of them.

## Tombstones and gc_grace_seconds
---

Delete one of the surviving rows:

```sql
DELETE FROM lab.events
WHERE id = c3e6ef8e-c884-4a7a-9b77-aeddfef8da60
AND ts = '2026-03-14 14:49:58.607Z';
```

```bash
bin/nodetool flush lab events
```

Dump the newest SSTable. You'll see:

```
{
  "partition" : { "key" : [ "c3e6ef8e-c884-4a7a-9b77-aeddfef8da60" ], "position" : 19 },
  "rows" : [
    {
      "clustering" : [ "2026-03-14 14:49:58.607Z" ],
      "deletion_info" : {
        "marked_deleted" : "2026-03-24T06:34:38.595658Z",
        "local_delete_time" : "2026-03-24T06:34:38Z"
      },
      "cells" : [ ]
    }
  ]
}
```

`"cells": []` - no data, just a death marker. Two fields command the tombstone:

- **`marked_deleted`**: the logical deletion timestamp (`markedForDeleteAt`). Any cell with a timestamp <= this value at the same clustering address is considered dead.
- **`local_delete_time`**: the wall-clock time the DELETE was written. This is the GC clock: `local_delete_time + gc_grace_seconds` = the earliest moment compaction is allowed to purge this tombstone.

The clustering value is still there, tombstones need an address so that they can shadow the right row across SSTables.

Now trigger compaction with the default `gc_grace_seconds = 864000` (10 days):

```bash
bin/nodetool compact lab events
```

The tombstone row is still there. Since, `local_delete_time` is seconds/minutes old and not 10 days old, compaction keeps it.

> This is intentional. In a multi-node cluster, a DELETE only reaches the nodes that are alive at that moment. `gc_grace_seconds` buys 10 days for any temporarily down replica to come back and receive the tombstone via repair.
>
> If we purged tombstones immediately, a replica that missed the DELETE could resurrect the row the next time it's queried, resulting in silent data resurrection.

Now let's force the purge by manipulating `gc_grace_seconds`:

```sql
ALTER TABLE lab.events WITH gc_grace_seconds = 0;
```

```bash
bin/nodetool compact lab events
tools/bin/sstabledump \
  ./data/data/lab/events-1b255f4def2540a60000000000000005/pa-*-Data.db
```

The tombstone row is gone. The partition for `c3e6ef8e...` no longer appears in the dump. The only row it ever had was at clustering `"2026-03-14 14:49:58.607Z"`, which is exactly what the DELETE targeted.

Compaction did three things in one pass: merged the SSTables, checked `local_delete_time + gc_grace_seconds`, purged the tombstone, and reclaimed the space.

Reset `gc_grace_seconds` back before moving on:

```sql
ALTER TABLE lab.events WITH gc_grace_seconds = 864000;
```

## Conclusion
---

You can now reproduce any SSTable behaviour from scratch, no longer a black box:

- A read burns at most one `Index.db` I/O on first access, then zero on every repeat (key cache)
- Compaction is a `K-way merge-sort` with a timestamp comparator. `Cells.reconcile()` is where LWW actually happens
- A tombstone is just a row with a timestamp and no cells. It shadows its target across SSTable boundaries during reads, but only at its own clustering address
- `gc_grace_seconds` is not an optimisation, it's a correctness window: purge too early and you get silent data resurrection on rejoining replicas

If [<u>Part 1</u>](/cassandra-sstables-whats-really-on-disk/) taught us what an SSTable is made of, **Part 2** is how we reason about what happens to that data at read time and across compaction.

I hope you found this somewhat interesting and useful. Until next time!
