---
layout: post
title: "into the eye - cassandra sstables - the flush path"
tags: [cassandra, storage-engine, sstables, memtable, internals]
---

This is **Part 3** of the SSTable series. [Part 1](/cassandra-sstables-whats-really-on-disk/) covered file layout and row encoding. [Part 2](/cassandra-sstables-the-read-path-and-compaction/) covered the read path and compaction.
In this post, we follow data from the moment it lands in memory to the moment it becomes an SSTable on disk.

## The Memtable: sorted by design

Writes don't go to disk immediately. They land in a **Memtable**, an in-memory collection of partitions, backed by a `ConcurrentSkipListMap`. In `<cassandra-package>/db/memtable/SkipListMemtable.java` this is declared as:

```java
ConcurrentNavigableMap<PartitionPosition, AtomicBTreePartition> partitions
```

Two things matter here. `Concurrent` means writes don't block each other. Multiple threads write into the same memtable without coordination. `NavigableMap` means partitions are kept in token order at all times.

That token order is deliberate. Flush has to write partitions in token order to produce a valid SSTable because `Index.db` and `Summary.db` assume sorted order to allow binary search. A scan of an unsorted index would degrade from `O(log n)` to `O(n)`. The `ConcurrentSkipListMap` gives you sorted iteration for free at flush time, with no sorting step needed.

## What triggers a flush

Four things can trigger a flush:

```
MEMTABLE_LIMIT           |   heap or off-heap pool pressure
COMMITLOG_DIRTY          |   oldest commit log segment needs recycling
MEMTABLE_PERIOD_EXPIRED  |   memtable_flush_period_in_ms timer expired
USER_FORCED              |   nodetool flush
```

`COMMITLOG_DIRTY` is worth understanding. Commit log segments are recycled when they fill up, but a segment can only be recycled once every table it covers has been flushed to disk. If the memtable hasn't been flushed, the segment still holds the only durable copy of that data. So a full segment puts back-pressure on any unflushed memtable: either flush now or can't make room for newer writes.

For `MEMTABLE_LIMIT`, Cassandra doesn't flush blindly. A background `MemtableCleanerThread` (in `<cassandra-package>/db/memtable/AbstractAllocatorMemtable.java`) watches total heap usage. When it crosses the threshold, it scans all live memtables, picks the one consuming the largest fraction of the pool limit, and signals only that table to flush.

## Little hands-on

Before triggering a flush, check the data directory - no SSTable files yet:

```bash
ls data/data/lab/events-*/
```

Insert a few rows and flush:

```sql
USE lab;
INSERT INTO events (id, ts, data) VALUES (uuid(), toTimestamp(now()), 'first');
INSERT INTO events (id, ts, data) VALUES (uuid(), toTimestamp(now()), 'second');
INSERT INTO events (id, ts, data) VALUES (uuid(), toTimestamp(now()), 'third');
```

```bash
bin/nodetool flush lab events
```

Check again, should see the full set of component files appears at once:

```bash
ls data/data/lab/events-*/
pa-1-big-CompressionInfo.db
pa-1-big-Data.db
pa-1-big-Digest.crc32
pa-1-big-Filter.db
pa-1-big-Index.db
pa-1-big-Statistics.db
pa-1-big-Summary.db
pa-1-big-TOC.txt
```

All eight components at once. That's the flush completing atomically, not files trickling in one by one.

## Swapping the memtable without stopping writes

When a flush is triggered, Cassandra doesn't pause writes. It creates a fresh empty memtable, swaps it in and new writes start going there immediately. The old memtable is sealed, but a write that started a millisecond before the swap might still be finishing up inside it.

> The question is:
>
> How does the flusher thread know it's safe to start reading the old memtable's data? That all writes to that old memtable is completed?

The answer is `OpOrder.Barrier` in `<cassandra-package>/utils/concurrent/OpOrder.java`. Every write thread calls `order.start()` when it begins, which registers it into the current `Group` - an atomic integer counter. The flusher thread calls `barrier.issue()` to expire that Group (no new writes can register) and then `barrier.await()`, which sleeps until the counter drains to zero. <i>Very elegant solution!</i>

```java
// write thread: knows nothing about flush
try (Group opGroup = order.start()) { // Register to group
    // do write work
}   // close() decrements the counter

// flush thread
barrier.issue();   // expire current group, new writes go to next group
barrier.await();   // wait for in-flight writes to drain
```

No lock anywhere. Write threads don't coordinate with the flush thread at all, they just `start()` and `close()` around their work. The barrier passively waits.

Alternatively if we had used locking to stop writes during a flush, it would be catastrophic. A flush writes potentially gigabytes to disk and that can take seconds. A lock held that long would stall every write to the table for the duration of every flush. Under any write load that would choke the system entirely.

## Writing to disk: the bridge moment

Once the barrier clears, the flush logic runs in `<cassandra-package>/db/memtable/Flushing.java`. The core of `writeSortedContents()` is quite simple:

```java
for (UnfilteredRowIterator partition : toFlush) {
    writer.append(partition);
}
```

`toFlush` iterates the memtable's `ConcurrentSkipListMap` in token order. Each element is an `UnfilteredRowIterator`, the exact streaming interface from the in-memory data model. The writer on the other side is `BigTableWriter` in `<cassandra-package>/io/sstable/format/big/BigTableWriter.java`, producing `Data.db` and `Index.db` exactly as described in Part 1.

This is where the two layers connect. The full in-memory data model: partitions, rows, cells, tombstones - streams directly into the on-disk format with no intermediate transformation.

`Index.db` is written in the same pass, one entry per partition, recording the byte offset in `Data.db` at the moment each partition is written. `Summary.db` is built from the index as it's written. Bloom filter entries are added as each partition key is processed. All of it happens in a single sequential pass - which is also why the resulting SSTable is always consistent. There's no separate reconciliation step; the files are produced together.

## Finalizing and going live

After the last byte is written, the writer syncs all buffers to disk and finalizes the remaining components: `Statistics.db`, `Filter.db`, `CompressionInfo.db`, and `TOC.txt`. At this point the files are on disk, but the SSTable isn't queryable yet.

Making it visible happens in `replaceFlushed()` inside `<cassandra-package>/db/lifecycle/Tracker.java`. The `Tracker` holds a `View` - an immutable snapshot of all live memtables and SSTables. To update it, Cassandra:

1. Calls `setupOnline()` on `<cassandra-package>/io/sstable/SSTableReader.java`: bloom filter, index summary, and file handles are all initialized
2. Creates a new `View` with the new SSTable added and the old (flushing) memtable removed
3. Swaps it in with a single CAS

The swap is instantaneous. Reads that started before it finish against the old view. Reads after it see the new SSTable. The bloom filter being ready before the swap matters, a read that hits the new SSTable immediately after the swap must be able to use it.

## Reclaiming memory

The old memtable is now out of the live set, but it can't be freed yet. A read that started just before the CAS swap might still be scanning it.

Cassandra issues a second barrier - a **read barrier** this time - and waits for those in-flight reads to drain. Only then is `discard()` on `<cassandra-package>/db/memtable/Memtable.java` called, releasing the memory back to the pool.

Two barriers, one for each direction:

```
write barrier await    |   safe to read memtable data
write SSTable files    |   Data.db, Index.db, Filter.db ...
Tracker CAS swap       |   SSTable visible, memtable removed from live set
read barrier await     |   safe to free memtable memory
memtable.discard()     |   memory returned to pool
```

## Reading the flush with sstablemetadata

Run `sstablemetadata` against the file that was just produced:

```bash
tools/bin/sstablemetadata data/data/lab/events-*/pa-1-big-Data.db
```

Here's what came back from our lab run (will differ for your run):

```
SSTable Level: 0
Minimum timestamp: 03/14/2026 20:20:02 (1773499802547369)
Maximum timestamp: 03/25/2026 23:28:24 (1774461504791177)
SSTable min local deletion time: no tombstones (9223372036854775807)
Replay positions covered: {CommitLogPosition(segmentId=1773499376267, position=159942)=CommitLogPosition(segmentId=1773499376267, position=248979), ...}
totalRows: 7
EncodingStats minTimestamp: 03/14/2026 20:20:02 (1773499802547369)
EncodingStats minLocalDeletionTime: 09/22/2015 05:30:00 (1442880000)
```

A few things map directly to what we have just covered:

**`SSTable Level: 0` >** Level 0 is always where fresh SSTables land. They haven't been through compaction yet.

**`Replay positions covered` >** The exact commit log segments and byte offsets this SSTable covers. Once the SSTable is durable, those segments can be recycled. Three ranges here means writes to this SSTable came from three different commit log segments over its lifetime, exactly the `COMMITLOG_DIRTY` pressure we discussed.

**`EncodingStats minTimestamp` >** The delta encoding baseline from Part 1. Every cell timestamp in `Data.db` is stored as a delta from this value.

**`EncodingStats minLocalDeletionTime: 09/22/2015` >** This one looks odd on a table with no tombstones. `EncodingStats` tracks the minimum `localDeletionTime` to use as a delta baseline for tombstone times. Live rows carry `DeletionTime.LIVE` with `localDeletionTime = Integer.MAX_VALUE`, including that in the minimum would make delta encoding useless, so Cassandra skips it. With no actual tombstones, the minimum never gets set from real data and stays at the hardcoded floor in `EncodingStats.NO_STATS`: epoch second `1442880000`, that 2015 date. When a real tombstone is written, it displaces this default with its actual `local_delete_time`.

## Conclusion

The flush path does three things that are easy to get wrong mentally:

- It swaps the memtable without stopping writes: via a barrier, not a lock.
- It produces a fully consistent SSTable in a single sequential pass: all components written together, not reconciled after.
- It makes the SSTable visible and discards the memtable memory in two separate atomic steps: one barrier for safety, one CAS for visibility.

And that's it!

If Part 1 showed you what's in an SSTable, Part 3 is how it gets there.

Toodle-oo!
