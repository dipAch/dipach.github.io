---
layout: post
title: "into the eye - cassandra sstables - what's really on disk"
tags: [cassandra, storage-engine, sstables, internals]
---

SSTables are Cassandra's immutable on-disk storage units — the files you end up with after every flush and compaction.
If you want to navigate the storage engine code, or reproduce a regression without guessing, you need to be able to read these files.
We'll flush real data, inspect every component, and trace the row encoding down to the flags byte.

## The setup
---

Before we look at anything, let's produce a real SSTable to work with. Fire up a single node (IntelliJ or CCM) from project root, then in `cqlsh`:

```sql
CREATE KEYSPACE IF NOT EXISTS lab
  WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 1};

USE lab;

CREATE TABLE events (
  id   uuid,
  ts   timestamp,
  data text,
  PRIMARY KEY (id, ts)
);

INSERT INTO events (id, ts, data) VALUES (uuid(), toTimestamp(now()), 'first');
INSERT INTO events (id, ts, data) VALUES (uuid(), toTimestamp(now()), 'second');
INSERT INTO events (id, ts, data) VALUES (uuid(), toTimestamp(now()), 'third');
```

Force a flush so the data lands on disk (`again from project root`):

```bash
bin/nodetool flush lab events
```

Then find the files that were written (`also from project root`):

```bash
find ./data/data/lab/events-* -type f | sort
```

You should see something like:

```
./data/data/lab/events-1b255f4def2540a60000000000000005/pa-1-big-CompressionInfo.db
./data/data/lab/events-1b255f4def2540a60000000000000005/pa-1-big-Data.db
./data/data/lab/events-1b255f4def2540a60000000000000005/pa-1-big-Digest.crc32
./data/data/lab/events-1b255f4def2540a60000000000000005/pa-1-big-Filter.db
./data/data/lab/events-1b255f4def2540a60000000000000005/pa-1-big-Index.db
./data/data/lab/events-1b255f4def2540a60000000000000005/pa-1-big-Statistics.db
./data/data/lab/events-1b255f4def2540a60000000000000005/pa-1-big-Summary.db
./data/data/lab/events-1b255f4def2540a60000000000000005/pa-1-big-TOC.txt
```

Eight files for three rows. Let's understand what each one is.

## SS what?!
---

Every SSTable is a **group of component files** sharing a descriptor prefix:

```
{version}-{generation}-{format}-{Component}.db
```

For a `trunk` build the version string is `pa` (the current BigFormat version). Here's what each component does:

```
pa-1-big-CompressionInfo.db   block map: compressed offsets → uncompressed offsets
pa-1-big-Data.db              the actual rows, token-sorted
pa-1-big-Digest.crc32         checksum over Data.db; validated on open
pa-1-big-Filter.db            Bloom filter (loaded fully into memory at open time)
pa-1-big-Index.db             (partition key → byte offset in Data.db) entries
pa-1-big-Statistics.db        metadata: token range, encoding stats, schema snapshot
pa-1-big-Summary.db           downsampled Index.db (every 128th key, held in RAM)
pa-1-big-TOC.txt              plain list of component files for this SSTable
```

A point lookup touches these in order: **Filter.db → Summary.db → Index.db → Data.db**.
Each is a bail-out gate — a key absent from the Bloom filter never reaches `Index.db` at all.

Still with me?

## sstabledump && sstablemetadata — what they're telling you
---

Run both tools against the file you just produced:

```bash
tools/bin/sstabledump \
  ./data/data/lab/events-1b255f4def2540a60000000000000005/pa-1-big-Data.db

tools/bin/sstablemetadata \
  ./data/data/lab/events-1b255f4def2540a60000000000000005/pa-1-big-Data.db
```

Here's the condensed dump output from our lab run (one partition shown):

```
[
  {
    "table kind" : "REGULAR",
    "partition" : {
      "key" : [ "c3e6ef8e-c884-4a7a-9b77-aeddfef8da60" ],
      "position" : 19
    },
    "rows" : [
      {
        "type" : "row",
        "position" : 19,
        "clustering" : [ "2026-03-14 14:49:58.607Z" ],
        "liveness_info" : { "tstamp" : "2026-03-14T14:49:58.599939Z" },
        "cells" : [
          { "name" : "data", "value" : "first" }
        ]
      }
    ]
  }
]
```

A few things worth noticing immediately:

**Token-sorted, not insertion-ordered.** You inserted `first`, `second`, `third` — but the file order (on my setup) is, `first → third → second`. That's Murmur3 token order. Every SSTable is an immutable, token-sorted file. Compaction merges SSTables by merge-sorting on this order — that ordering invariant is something you'll bump into constantly in the compaction code.

**`position` = byte offset in Data.db.** `"position": 19` means that partition starts 19 bytes into `Data.db`. The first 19 bytes are the SSTable header encoding `EncodingStats`. `Index.db` stores `(key → position)` pairs so a read can seek directly. When you see `SSTableReader.getPosition()` in the code, this is the value it resolves.

**`ts` is not in `cells` — it's the clustering key.** `ts` is part of `PRIMARY KEY (id, ts)`, so it's encoded in the `Clustering` object, not as a `Cell`. Only non-PK columns appear in `cells`. That's why `column count = 1`, not 2.

**`liveness_info` is not a cell timestamp.** It's the row-level `LivenessInfo` — written when you do a plain `INSERT` without TTL. Notice the sub-second precision: `599939` microseconds vs the clustering's `607` milliseconds. Different clocks, different purposes.

Now the `sstablemetadata` output has two more gems:

```
SSTable min local deletion time: no tombstones (9223372036854775807)
```

`9223372036854775807` is `Long.MAX_VALUE`. In Cassandra, every deletion time field needs a sentinel value to mean "not deleted". Rather than adding a separate boolean flag, Cassandra uses `Long.MAX_VALUE` as the **LIVE sentinel** — a value so far in the future it will never be treated as a real GC timestamp. You'll find it defined as `DeletionTime.LIVE.localDeletionTime()` in `<cassandra-package>/db/DeletionTime.java`.

```
EncodingStats minLocalDeletionTime: 09/22/2015 05:30:00 (1442880000)
EncodingStats minTimestamp: 03/14/2026 20:19:58 (1773499798599939)
```

That `2015` date isn't a bug — it's a **delta baseline** hardcoded in `EncodingStats.NO_STATS`. Timestamps on disk are stored as deltas from this baseline to keep the vint small. We'll see exactly how in the next section.

## How a row is encoded on disk?
---

`<cassandra-package>/db/rows/UnfilteredSerializer.java` is the class that reads and writes every row to `Data.db`.

Every row on disk is laid out as:

```
<flags byte>
[<extended flags byte>]       only if bit 0x80 is set
<clustering>
<row_size vint>               SSTable only, not on-wire
<prev_unfiltered_size vint>   SSTable only
[<timestamp delta>]           if HAS_TIMESTAMP
[<ttl delta>]                 if HAS_TTL
[<local_deletion_time delta>] if HAS_TTL
[<row deletion time>]         if HAS_DELETION
[<columns subset bitmap>]     absent if HAS_ALL_COLUMNS
<cell data...>
```

The **flags byte** controls which optional fields follow it. These constants are defined as a block at the top of `UnfilteredSerializer`:

```
IS_MARKER       0x02  range tombstone marker, not a row
HAS_TIMESTAMP   0x04  row has a liveness timestamp
HAS_TTL         0x08  row has expiry info
HAS_DELETION    0x10  row has a row-level tombstone
HAS_ALL_COLUMNS 0x20  all schema columns present — no subset bitmap needed
EXTENSION_FLAG  0x80  a second flags byte follows
```

Your `"first"` row had `HAS_TIMESTAMP | HAS_ALL_COLUMNS` — no TTL, no deletion, one column present. That's a tight encoding.

Two design decisions here that matter:

**`row_size` enables O(1) skipping.** The `skipRowBody()` method in `UnfilteredSerializer`:

```java
public void skipRowBody(DataInputPlus in) throws IOException {
    int rowSize = in.readUnsignedVInt32();
    in.skipBytesFully(rowSize);
}
```

To skip a row during a compaction scan, Cassandra reads just the size vint and jumps. No clustering parsing, no cell parsing. The `position` values in your `sstabledump` output are meaningful precisely because the index points directly to a row, and skipping forward from there is cheap.

**`previousUnfilteredSize` enables backward scans.** Each row stores the size of the *previous* row on disk, written by the `serialize()` method in `UnfilteredSerializer`. To scan backwards (for `ORDER BY DESC` queries), Cassandra reads the previous size and seeks back — no separate reverse index needed.

> If you add a new field to the row format, you must handle both the SSTable path (with `row_size`) and the on-wire path (hints, streaming, without `row_size`). The divergence is an `isForSSTable()` branch inside `serialize()`.

## SerializationHeader — the schema snapshot
---

**`SerializationHeader`** (`<cassandra-package>/db/SerializationHeader.java`) is a schema snapshot baked into the SSTable at write time. It lives in `Statistics.db` and holds:

- `keyType` — how to deserialize the partition key bytes
- `clusteringTypes` — one type per clustering component
- `columns` — the full column list at write time
- `stats` — `EncodingStats` (the delta baselines we saw in `sstablemetadata`)

This is the bridge between an immutable file and a mutable schema. When you `ALTER TABLE`, the live schema changes — but old SSTables still need to be readable.

**Delta encoding** — the `writeTimestamp()` and `readTimestamp()` methods in `SerializationHeader`:

```java
public void writeTimestamp(long timestamp, DataOutputPlus out) throws IOException {
    out.writeUnsignedVInt(timestamp - stats.minTimestamp);
}

public long readTimestamp(DataInputPlus in) throws IOException {
    return in.readUnsignedVInt() + stats.minTimestamp;
}
```

Timestamps in a single SSTable are usually close together, so the delta is small — fits in 1–2 bytes as a vint instead of always 8. Same pattern for TTL and `localDeletionTime`. This is the `EncodingStats minTimestamp: 1773499798599939` you saw. The baseline is the smallest timestamp across the SSTable, or across all input SSTables during compaction.

**Dropped column handling** — the `toHeader()` method in `SerializationHeader.Component`:

```java
ColumnMetadata column = metadata.getColumn(name);
if (column == null || column.isStatic() != isStatic) {
    column = metadata.getDroppedColumn(name, isStatic);
    if (column == null)
        throw new UnknownColumnException(...);
}
```

When loading an old SSTable whose column was since `DROP`ped, Cassandra creates a fake `ColumnMetadata` from the dropped column registry. The cell data is read and silently discarded. Without this, a single `ALTER TABLE DROP COLUMN` would make every old SSTable unreadable.

The full chain on read:

<img src="/images/sstable-read-chain-repr.png" alt="SSTable read chain" width="500" height="150"/>

If you ever add a time-based field to a row, you'd add a `write*/read*` pair here, update `EncodingStats`, and bump the SSTable version string.

## Conclusion
---

After flushing three rows and poking the files with two tools, you can now open any SSTable without them:

- The file naming tells you the format version (`pa`), generation, and format (`big`)
- Token order tells you where a key is relative to others — and why `Data.db` is never a simple hash map
- The flags byte tells you exactly which optional fields follow the clustering in each row
- The `sstablemetadata` output maps directly to `SerializationHeader` and `EncodingStats` in the code

In Part 2 we'll do the read path — watching all five lookup gates fire live with `TRACING ON` — and then the full compaction lifecycle: two SSTables in, one out, with LWW cell reconciliation and tombstone purge hands-on.

Happy learning!
