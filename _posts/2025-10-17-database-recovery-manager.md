---
layout: post
title: "database recovery manager - The (mostly) important points"
tags: [database, internals, recovery-manager]
---

This is a continuation of Edward Sciore's text on "Database Design and Implementation". This post summarizes the mechanics of the "Recovery Management" component
in a database system. It tries to capture the key aspects and design principles that drive the operational guarantees provided with respect to data correctness and prevention of data corruption.

## What is a Recovery Manager?
---

Recovery manager's sole responsibility lies in bringing the database system to a logically sane state in the course of both normal and abnormal operations. It uses the means of a transaction log to verify what changes will stay and which ones need to be undone.

In order to perform it's job with clarity, it is responsible for appending any transaction's log/activity records as well. These records guide it as to which transaction's modifications stay Vs the ones that needs reverting to an accepted previous state.

Recovery manager can be called on demand by a participating transaction (commit/rollback delegation) or it could be invoked as part of the database startup ritual (after a crash or even during first boot).

## Key Responsibilities
---

As a core component in upholding database safety, it majorly encompass below tasks:

* **Manage Transaction Log:**

We can have mainly below categories of loggable work,
    - Start of a transaction
    - Commit or Rollback of a transaction (Marking transaction complete stage)
    - Value update action by a transaction

    Eg:
    ```
    <START, tx1>
    <COMMIT, tx1>
    <START, tx2>
    <START, tx3>
    <SETINT, tx2, table1, blk1, offset, oldval, newval>
    <COMMIT, tx2>
    <SETINT, tx3, table1, blk1, offset, oldval, newval>
    <ROLLBACK, tx3>
    ```

* **Transaction Rollback:**

The recovery manager can be called to rollback a specified transaction Tx. In order to rollback, the recorvery manager needs to undo all of
Tx's modifications (if any).

The recovery manager needs to follow the update category of log records for the transaction, which contains the details of the modifications. It needs to restore
the old values for each update operation.

Also the recovery manager reads the log from the end and not from beginning. We move from end of log to the record that marks the start of the transaction Tx. All update records for Tx will be reverted as we go backwards.

* **System Recovery:**

This basically involves the Undo-Redo recovery maneuver. We want to undo, incomplete transaction's work and we want to make committed transaction's work durable.

The recovery manager starts from the bottom of the log, i.e., from the most recent log record and goes backwards/upwards.

Once the log records are scanned till the top (Phase 1), it starts moving downward now for its second pass to perform the redo operation (Phase 2).

This process is done during database system startup to enforce correctness in terms of atomicity (revert incomplete or half done work) and durability (persist what was deemed complete to the client).

## Recovery Variants
---

Let's have a quick glance at the possible recovery types,

### Undo-Only Recovery

In this type of recovery, we are not worried about the Redo action of the Recovery Manager. How is that facilitated? Well, firstly for that we have to be sure that
committed transactions have their data buffers already flushed to disk. This gives us the confidence of durable operations (remember the D in `aciD`?)

Undo-Only recovery results in faster progress. Why? Because it requires only one pass through the log file to perform the Undo operation on incomplete transactions.

Recall that we needed transactions to add the new value for an update log entry for the Redo operation. But if we don't have to do a "Redo" operation, then we may skip logging that, right? Hence the update log entry also doesn't need to capture the new value and the log size is reduced by some extent across transactions and modifications. We still need to log the old value, as that is needed for the "Undo" operation.

But transaction commit takes a hit, as before appending a commit entry to the log, the data buffers modified by the transaction needs to be flushed to disk.

So what are the steps:
```
- Flush the data buffers modified by transaction that has committed.
- Write the commit log entry for transaction to log page.
- Flush the log page to current disk block of log file, for persisting the log entry.
```

And we are done. Simple, right?

### Redo-Only Recovery

Hmmm ... what do you think we do in this? We basically try to avoid doing any "Undo" operations. We only aim at asserting committed transactions remain durable and their modifications are not lost, in case there is any disgraceful (too strong a word, but still) shutdown.

But how do we ensure that we don't need to do any undo operations? Well for starters, we don't allow writing/flushing buffers until a transaction completes (commit/rollback). Basically keep the buffer pinned.

Recall either we explicitly flush a data buffer (on comit/rollback) or it gets flushed when replacing with a new disk block (provided it is unpinned). The buffer manager does not let any pinned buffer to be replaced or associated to a new disk block.

Well it is faster as we only care about transactions that have a committed log entry. Ignores other transactions that are incomplete or rolledback.

On the not so bright side though, this will increase buffer contention as transactions might hold onto modified buffers pinning them for longer than usual.

## Write-Ahead Logging (WAL)
---

Just a fancy term that tells what was most likely modified in a durable fashion. Whether the actual modification reached the disk or not is secondary.

Why is this necessary??

In order to undo any incomplete transaction's modifications, we need to know what and where the modification happened. Without that information the recovery manager is helpless.

WAL helps the recovery manager to know what changes (if any) were done by incomplete transactions and needs reverting. Imagine the case, where a change from an incomplete transaction makes it to disk. But there is no log entry specifying that, then there is no way for the database to revert the modification. Hence, data corruption creeps in and we (the recovery manager) can do nothing about it. Sad state of affair, must say.

Hence, the concept of WAL says that before flushing any modification in the data buffer to disk, one must write and flush the log entry for that modification.

The log entry has a corresponding LSN (Log Sequence Number) that the buffer keep tracks of it internally. When a flush request comes in, it makes the log manager flush up until the current LSN tracked by the buffer, marking changes/modifications up until the LSN as pushed to disk.

## Checkpointing
---

> What is the objective?
- We want to know where in the log we can stop scanning further backwards. This needs guarantee that after a certain point, all records going back belong to completed transactions. So, no need of "Undo" activity on those transactions (**Atomicity Guarantee**).
- For all such transactions, the data buffers were flushed to disk and we don't have to engage in a "Redo" activity (**Durability Guarantee**).

This concept vastly reduces the work needed to be done by the recovery manager and also the amount of past data in log records that needs maintaining.

There are 2 categories of checkpointing explained in the text:

1. **Quiescent Checkpointing**

    This is basically a stop the world checkpointing. In this method, when the checkpointing process kicks in, it stops accpeting any new transactions and waits for already running transactions to complete. Once running transactions complete and their modified buffers are flushed, a checkpoint log record is added and then the log is flushed. Normal operation resumes post that.

    Usually performed at startup when recovery is done, so that only new log records post the checkpoint marker are to be considered by the recovery manager in the future.

2. **Non-Quiescent Checkpointing**

    So the problem with previous checkpointing method (**Quiescent**), was that the database was rendered unresponsive till the checkpointing completes. This could have serious performance implications for high traffic database nodes.

    To overcome this, we have another method that keeps track of the running transactions when the checkpoint process started. It records the list of transactions as part of its checkpoint log record.

    This method also stops accepting new transactions albeit for a short period only, as this method doesn't have to wait for running transactions to complete.

    Basically the recovery manager relies on finding the earliest transaction's start log record in the checkpoint record's transaction list, so that it can stop scanning backwards any futher post that.

## Conclusion

Recovery management might seem like one of those behind-the-scenes components that you don't think about until something goes wrong. But as we've seen, it's the safety net that keeps your database from turning into a chaotic mess when things inevitably go sideways.

The beauty of recovery managers lies in their simplicity of purpose—keep what's committed, undo what's not. Whether you're dealing with undo-only recovery for speed, redo-only for safety, or the full undo-redo approach for complete coverage, the core principle remains the same: use the log to maintain sanity.

Write-Ahead Logging isn't just a best practice—it's the foundation that makes recovery possible in the first place.

And checkpointing? That's what keeps your recovery process from having to replay the entire history of your database every time something crashes.

Next time your database comes back up after an unexpected shutdown and everything just works, you can thank the recovery manager doing its quiet, essential job in the background.

Hope this breakdown made the recovery manager's role a bit clearer. Until next time!
