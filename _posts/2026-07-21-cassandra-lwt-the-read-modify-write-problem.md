---
layout: post
title: "into the eye - cassandra lwt - the read-modify-write problem"
tags: [cassandra, lwt, paxos, linearizability, internals]
---

In this post we cover what LWTs offer, how they are implemented using Classic Paxos and later the internal mechanics of the conditional mutation path.

This article also introduces a nodetool tracing utility that allows us to view an actual paxos consensus round triggered by an LWT. It does a step-by-step breakdown of the phases and the actual outcome recorded at each step, which closely relates to the theory explained in the text. This tool gives the necessary visual insight into the actual pieces of the algorithm and prevents relying on assumed mental models about how this is implemented or behaves in the wild.

Briefly, LWTs are a Cassandra write feature providing optimistic concurrency control using a *compare-and-swap (CAS)* mechanism. They ensure atomicity, isolation, and linearizable consistency for conditional updates, but only within a **single partition**.

> Why just a single partition though?
> ---
> The limitation is architectural and what cassandra tries to avoid. Cassandra's LWT runs one independent Paxos instance per partition. The ballot, the promise history, the in-flight state in system.paxos internal table, all of it is scoped to a single <partition-key, table>. There is no global ballot space, no shared consensus state that spans partitions.
>
> That design works perfectly for single-partition atomicity. A coordinator wins a ballot for partition A, reads it, proposes a write, commits. Any other coordinator that tries to touch partition A concurrently gets serialized by the ballot ordering. The partition becomes the unit of consensus.
>
> The problem with multi-partition writes is that we need all-or-nothing across partitions and if the write to one partition succeeds but to another fails, we have a half-committed state with no way to roll it back. Closing that gap requires a second coordination layer above Paxos that can make the commit decision for all partitions atomically.
>
> **Hint:** Think of 2PC drawbacks.

The feature relies on Paxos consensus protocol to coordinate agreements across participating nodes, at the cost of higher latency compared to regular writes.

Unlike other real-world Paxos deployments that employ a steady-state, traditional Cassandra LWT does not have a stable leader or a long-lived steady state. Every LWT can have a different proposer, and concurrent proposers may compete by issuing higher ballot numbers. So, the contention factor is high compared to steady leader-based setups.

## The cheap write
---

A regular Cassandra write is quite fast. The coordinator sends the mutation to enough replicas for the configured *<u>consistency level</u> (CL)*, waits for replica acknowledgements that satisfies the CL definition, and returns to the client. It employs a *<u>last-write-wins</u> (LWW) semantic*, which is simple to reason about in the face of conflicts.

But sometimes <u>last-write-wins</u> is just not acceptable.

## For lack of better example: The transfer problem
---

Say we want to protect a bank balance during the transfer / withdraw action:

```sql
SELECT balance FROM accounts WHERE id = 1;
-- balance is 1000
UPDATE accounts SET balance = balance - 200 WHERE id = 1;
-- deducted 200
```

This is a **read-modify-write** sequence. We take in the current value, compute a new value, and write it back. This brings about a possible **race window** between the read and the subsequent write. Imagine a concurrent transfer that reads the same balance of `1000` and deducts `300`, and performs a write will cause issues. Whoever writes last: **wins** and one transfer is silently overwritten. This is *BAD* bad!

> Quorum reads and quorum writes don't close the window. Why?
> ---
> Because quorum is a replication guarantee, not a serializability guarantee.

When we do a quorum read, the expectation is: *"I will read from enough replicas that I am guaranteed to observe the most recently written value."*

When we do a quorum write, the expectation becomes: *"I will write to enough replicas that the next quorum read will observe this written value."*

> **Both guarantees hold individually, but neither operation knows the other exists**. The read and the write are two separate requests that arrive at the coordinator at different moments.

Nothing in Cassandra's replication layer knows that the value `1000` we read and the value `800` we are about to write are part of the same logical operation.

There is possibility that another client can slip in between. It does its own *quorum read (also sees 1000)*, computes its *new value (700)*, and does its own *quorum write*, all before the initial concurrent write arrives. Both of them read 1000 and both of them write back a value derived from 1000. Whoever arrives at the replicas last: **wins**, and the first write is silently *overwritten as if it never happened*. Yet, both sequence's quorum guarantees held perfectly. The anomaly is in the gap between the operations, which quorum has no concept of tying together.

## Closing the gap, the window
---

This is what the `<IF>` clause adds to the mix:

```sql
UPDATE accounts SET balance = 800 WHERE id = 1 IF balance = 1000;
```

- When the statement returns `[applied] = true`: the balance was `1000` at the moment the write was applied and nothing changed it in between.
- Else, if something modified `balance` during the race window, Cassandra returns `[applied] = false` instead of silently overwriting.

Under the hood, Cassandra runs this through **Paxos**. The `<IF>` clause is the CQL trigger to perform a CAS based consensus round. Neat!

## Why Paxos?
---

Paxos solves the coordination problem. It ensures that across all replicas, at most one value can be chosen per "slot", i.e., one winner per round. Concurrent coordinators either win or learn they lost, and if they lost they see the winner's value before retrying.

The mechanism is the **ballot**: a monotonically increasing identifier *(a time-based UUID)* that each coordinator generates. Replicas remember the highest ballot they have ever *seen (or promised)*. If a coordinator sends a ballot that is lower than what a replica has already promised to someone else, the replica rejects it and returns the higher ballot.

The losing coordinator must retry with an even higher ballot. This ordering is how concurrent coordinators are serialized without a central lock.

Could we have used another consensus algorithm, say *Raft or VSR? Very well,* give it a go! ~ But the core ideology being, we need something that allows for agreeing on an order.

## Ticket please? Tale of four (চাৰিটা) round trips
---

LWT takes a minimum of four round trips whereas a regular write takes one:

<div><img src="/images/paxos-round-trip.png" alt="Paxos round trip"></div>

Each step requires a quorum of replicas before the coordinator moves forward. If any step fails, say a replica rejects the ballot or a deadline exceeds, then the coordinator retries from the Prepare step with an even higher ballot.

Let's briefly go over the phases and the sub-steps:
> **STEP 1:** PREPARE and occasional RE-PAIR, rhymes sort of?

The gateway to performing a Paxos consensus round can be seen inside the method definition: `<cassandra-package>/service/StorageProxy.java`**#**`doPaxos()`

Firstly, a *request deadline duration* is computed so that the round doesn't keep on *(re-)trying* indefinitely. Now within the bounds of the declared deadline, the implementation first checks for replica liveness and establish the required quorum to satisfy the linearization agreement.

If the required replicas are up and alive, then we begin the paxos session. Check definition, `<cassandra-package>/service/StorageProxy.java`**#**`beginAndRepairPaxos()`

Now this step basically does a few additional things that is not in the *<u>fast-path</u>* of the algorithm. Let's gloss over them:
- We again compute a preparation phase deadline and generate a ballot that we will use to propose our mutation.
- Once we have generated a candidate ballot, we kick-off the "Prepare" messages over to the respective replica nodes. Once messages are sent, a `CountDownLatch` is used to wait on the majority responses.
- Based on the quorum responses, if we get to know that a higher ballot was promised, we abort and re-try.
- Here's the interesting bit ~ If there is **no problem with our ballot**, and if any *`in-progress mutation exists with higher ballot than the most recent commit, we first complete the in-flight mutation before anything else`*.
  - Imagine this as a passing the baton in a marathon relay race kind-of action. We complete someone else's leftover bit ourselves before getting on with our own.
  - This is a great way to preserve order, as we are not blocking the in-progress mutation with our own ballot, which would've needed the in-progress mutation to start-over with a newer and higher ballot than ours, leading to more contention.
  - In this case, we will again need to retry with a new ballot as we sort-of donated our ballot to the previous in-flight mutation.
  - Another case is that the responded quorum of replicas have not learned about the most recent commit.
    - This doesn't mean the commit was not observed my majority. It just means the quorum that responded, consists of nodes that are yet to learn about the most recent accept.
    - So we perform a repair operation and propagate the decision to those before continuing with our own mutation.
    - This again warrants a new fresh ballot for our original mutation.

Once the ballot is promised to, we can proceed with the next phase. Let's go!

> **STEP 2:** Pre-condition check ~ `<IF>` CQL clause

This step reads the current value for the target of the <IF> clause and ensures that it satisfies the predicate. It issues a single-partition read command with the required read consistency level needed for paxos based CAS, and usually set to SERIAL consistency level.

If all is well then we get the set of updates scoped to a single parition and proceed to the next step. The updates might be augmented with trigger induced updates if any trigger conditions match w.r.t the original mutation(s).

> **STEP 3:** PROPOSE

This follows similar suit as the "Prepare" step. It sends the mutation to the replicas and waits for response from a quorum. If proposal was not accepted by a majority, then it will perform a bounded back-off (basically sleep for maximum of 100 ms) and then retry, provided request deadline is not reached.

The definition for this step can be found at: `<cassandra-package>/service/StorageProxy.java`**#**`proposePaxos()`

If proposal is accpeted by a quorum (hurray!), then we move to the next step which is to commit the proposal and make it permanent.

> **STEP 4:** COMMIT

Sends the commit paxos messsage to the replicas and may or may not wait for acknowledgements. If any replicas are down, it will store mutation hints to later perform a hinted hand-off.

The definition for this step can be found at: `<cassandra-package>/service/StorageProxy.java`**#**`commitPaxos()`

And just like that we are done with the round. Quite the *theatrics!*

## Lab time: Hands on
---

### Cluster setup

Let's get down to business. We will spawn a three-node CCM cluster, `SimpleStrategy` with `replication_factor=3`:

```bash
ccm create lwt-lab --install-dir=/path/to/cassandra -n 3
ccm start
```

Schema and seed data:

```bash
ccm node1 cqlsh -e "
  CREATE KEYSPACE ks
    WITH replication = {'class': 'SimpleStrategy', 'replication_factor': 3};
  CREATE TABLE ks.accounts (id int PRIMARY KEY, balance int);
  INSERT INTO ks.accounts (id, balance) VALUES (1, 1000);
  INSERT INTO ks.accounts (id, balance) VALUES (2, 500);"
```

The seeds used a plain `INSERT` ~ No `IF`, nor Paxos or any sort of coordination. Just plain old replication.

### Running LWT

We will try `IF NOT EXISTS` clause on a key that already exists, then on a new one:

```bash
ccm node1 cqlsh -e "INSERT INTO ks.accounts (id, balance) VALUES (1, 9999) IF NOT EXISTS;"
```
```
 [applied] | id | balance
-----------+----+---------
     False |  1 |    1000
```

```bash
ccm node1 cqlsh -e "INSERT INTO ks.accounts (id, balance) VALUES (99, 250) IF NOT EXISTS;"
```
```
 [applied]
-----------
      True
```

The result tells us whether the condition held. What we do not see is that both statements triggered a full Paxos round. Let's look inside.

### Utility: nodetool `<paxostrace>`

In order to better understand what actually happens in terms of the consensus steps, I wanted for myself to be able to trace a round across ballot retries. So, whipped up a trace utility that gives visibility to it and hopefully aids in learning and making the steps concrete with a real-time lab workload, rather than just assuming a mental model that might be right.

<u>paxostrace</u> is a `nodetool` sub-command built on top of a custom JMX store that instruments every Paxos phase on every node. It collects events from all nodes, correlates them by ballot UUID, and prints a single stitched cross-node timeline per round. A round here means one client LWT call; each round may span multiple ballot attempts if Paxos retries due to contention.

We can run it against our table after any client LWT invocation as below:

```bash
nodetool paxostrace \
  --table ks.accounts \
  --nodes 127.0.0.1:7100,127.0.0.1:7200,127.0.0.1:7300 \
  --limit 2 \
  --latency
```

> `--nodes` takes a comma-separated list of `host:jmx_port` for each node. Probe for the JMX ports using command: `ccm nodeN show | grep jmx_port`.
>
> `--limit` caps how many rounds are shown, most-recent first. So --limit 2 shows the 2 most recent LWT calls (rounds), regardless of how many ballot attempts each contains.

We will talk more about the tool a little later and you can dig-in further if interested.

### Making sense of the trace: [applied] = true

```
Round 1  [roundId=77908370]  19:14:11.139  COMMITTED  (1 ballot attempt(s))
  Ballot ec24d622
    19:14:11.139  REPLICA      /127.0.0.1:7000  LEGACY_PREPARE  outcome=PROMISED
    19:14:11.139  REPLICA      /127.0.0.3:7000  LEGACY_PREPARE  outcome=PROMISED
    19:14:11.140  COORDINATOR  /127.0.0.1:7000  PREPARE_DONE    outcome=PROMISED
    19:14:11.140  REPLICA      /127.0.0.2:7000  LEGACY_PREPARE  outcome=PROMISED
    19:14:11.144  REPLICA      /127.0.0.1:7000  LEGACY_PROPOSE  outcome=ACCEPTED
    19:14:11.144  COORDINATOR  /127.0.0.1:7000  PROPOSE_DONE    outcome=ACCEPTED
    19:14:11.144  REPLICA      /127.0.0.2:7000  LEGACY_PROPOSE  outcome=ACCEPTED
    19:14:11.144  REPLICA      /127.0.0.3:7000  LEGACY_PROPOSE  outcome=ACCEPTED
    19:14:11.145  REPLICA      /127.0.0.1:7000  COMMIT
    19:14:11.145  COORDINATOR  /127.0.0.1:7000  COMMIT_DONE
    19:14:11.145  REPLICA      /127.0.0.2:7000  COMMIT
    19:14:11.146  REPLICA      /127.0.0.3:7000  COMMIT
  Latency:  prepare=~1ms  check=~4ms  propose=~0ms  commit=1ms  total=7ms
```

The round header tells you the outcome (`COMMITTED`), a `roundId` that groups all ballot attempts for one LWT call, and the wall-clock time of the first event. Each line below it is one event from one node, sorted by timestamp.

- **LEGACY_PREPARE / PROMISED:** The replica received ballot `ec24d622` and promised not to accept any lower ballot. All three replicas promised and thus coordinator reached quorum.
- **PREPARE_DONE / PROMISED:** Coordinator's view: promise quorum secured. It reads the current row under the ballot's protection and evaluates the `IF` condition.
- **LEGACY_PROPOSE / ACCEPTED:** Each replica accepted the proposed write and persisted it to `system.paxos`. When a quorum accepts the proposal, coordinator moves to commit.
- **PROPOSE_DONE / ACCEPTED:** Quorum of accepts reached.
- **COMMIT:** Each replica applied the mutation to the actual table and cleared its Paxos state.
- **COMMIT_DONE:** Commit quorum is reached and round is now complete.

All in all twelve events observed, comprising of three fan-outs (Prepare, Propose, Commit) and each requiring a quorum reply before the coordinator advances.

The `Latency:` line (produced by `--latency`) breaks down where the 7ms went.
- `prepare=~1ms` is the Prepare fan-out: messages out, promises in and quorum reached.
- `check=~4ms` is the largest slice. This is the gap between `PREPARE_DONE` and the first `LEGACY_PROPOSE`, the window where the coordinator read the row under ballot protection and evaluated `IF balance = 1000`, built the write mutation and dispatched propose messages.
- `propose=~0ms` and `commit=1ms` are sub-millisecond: once the mutation was ready, replicas accepted and committed fast. Metrics (such as, `prepare`, `propose` & `commit`) that span node boundaries, might carry clock-skew noise. The `"check="` metric is coordinator-only and exact. Both `PROPOSE_DONE` and `COMMIT_DONE` are emitted by the coordinator on the same node.

### Analysing the trace on: [applied] = false

Now, let's look at an interesting bit. Initially I thought if a pre-condition fails, we just bail. There's nothing to do. Right? **Nope, that's not what happens**. Mind-boggling!

Firstly, let's absorb the below LWT trace for a moemnt:

```
Round 1  [roundId=425bb584]  19:14:25.699  ACCEPTED (commit skipped)  (1 ballot attempt(s))
  Ballot f4d2ab32
    19:14:25.699  REPLICA      /127.0.0.1:7000  LEGACY_PREPARE  outcome=PROMISED
    19:14:25.699  REPLICA      /127.0.0.2:7000  LEGACY_PREPARE  outcome=PROMISED
    19:14:25.699  REPLICA      /127.0.0.3:7000  LEGACY_PREPARE  outcome=PROMISED
    19:14:25.700  COORDINATOR  /127.0.0.1:7000  PREPARE_DONE    outcome=PROMISED
    19:14:25.701  REPLICA      /127.0.0.1:7000  LEGACY_PROPOSE  outcome=ACCEPTED
    19:14:25.701  COORDINATOR  /127.0.0.1:7000  PROPOSE_DONE    outcome=ACCEPTED
    19:14:25.701  REPLICA      /127.0.0.2:7000  LEGACY_PROPOSE  outcome=ACCEPTED
    19:14:25.701  REPLICA      /127.0.0.3:7000  LEGACY_PROPOSE  outcome=ACCEPTED
  Latency:  prepare=~1ms  check=~1ms  propose=~0ms  total=2ms
```

Now let's dissect this and interpret.

The Prepare phase is identical, the algorithm has no way to short-circuit it. The condition check happens *after* `PREPARE_DONE`, when the coordinator reads the row under the ballot and finds `id=1` already exists. The condition fails, so the proposal instead becomes an **empty update**.

The replicas still ACCEPT it, they do not know or care whether the mutation is empty. What changes is at the coordinator: `commitPaxos()` is never called at all and no commit message is sent. This is safe because `beginAndRepairPaxos()` also skips replaying empty in-progress updates, so there is no correctness reason to ever commit an empty proposal. That is what `ACCEPTED (commit skipped)` means in the round header, and why there are no COMMIT events in the trace.

But this means there is some cost involved, as we are not doing anything valuable from our gain-oriented perspective.

Speaking in terms of the cost we paid: two full fan-outs (Prepare × 3, Propose × 3) regardless of the outcome. **LWT that returns `[applied] = false` is not cheaper than the one that returns `[applied] = true`**.

The condition check is not a pre-flight: it runs inside the Paxos round, after the ballot is won.

> The latency breakdown surfaces something concrete: `check=~1ms` in not applied Vs `check=~4ms` in the successful case.

When the condition fails, the coordinator reads the row and finds `id=1` already exists and immediately proposes an empty mutation. There is nothing to build in this case.

But, when the condition succeeds, the coordinator computes the actual write, which takes the extra time. No `"commit="` metric appears in the latency line-item because `commitPaxos()` is skipped entirely for empty updates and no `COMMIT_DONE` event is emitted, so there is no end-timestamp to compute the delta against.

`total=2ms` Vs `total=7ms` in this instance, shows how much the synchronous commit phase contributes when it runs.

<div style="display: inline-block; border: 2px solid black; padding: 5px; width: 250px; margin-top: 30px; margin-bottom: 30px;"><img src="/images/ace.jpeg" alt="Understood"></div>

## How `<paxostrace>` works? (skippable)
---

The tool is simple and it is not reading any log files or scraping anything after the fact. Events are captured **live, inside the Paxos state machine**, at the exact points where each phase outcome is decided.

> The modifications are available in my Cassandra fork, under the branch:
<a href="https://github.com/dipAch/cassandra/tree/lwt-trace-tool" target="_blank">`lwt-trace-tool`</a>

**Instrumentation:** `PaxosState.java` is the per-replica state machine. It has three core methods: `promiseIfNewer()`, `acceptIfLatest()`, and `applyCommit()`, each with several return sites depending on the outcome (PROMISE, REJECT, PERMIT_READ, ACCEPTED, etc.). A call to `PaxosTraceStore.add()` is inserted at each of those sites, capturing the outcome as it happens, not reconstructed, nor approximated. On the coordinator side, `StorageProxy.doPaxos()` emits its own events (`PREPARE_DONE`, `PROPOSE_DONE`, `COMMIT_DONE`) after each quorum reply comes in.

**In-memory store:** Events are written into `PaxosTraceStore`, a singleton that lives in the JVM and is always on. Currently not tunable via means of enabling a tracing session, or by flipping any flag. The store keeps a `ConcurrentHashMap` keyed by `"keyspace.table"`, with a bounded `ArrayDeque` per table (50k events total, evicting oldest per table when full). Each event is stored as an immutable value object: node address, role (COORDINATOR or REPLICA), phase label, ballot UUID, partition hex, token, wall-clock timestamp, and optional fields like `outcome` and `supersededBy`.

**JMX transport:** The store exposes a `PaxosTraceMBean` interface over JMX. On running `nodetool paxostrace`, it opens one `NodeProbe` connection per node, hitting each node's JMX port independently and calls `getEvents("ks.table")` on each. Events come back as a `List<String>` of JSON lines (no custom serializable types on the wire, so no classpath problems). The tool merges all events client-side, sorts by timestamp, and groups them.

**Cross-node stitching:** The ballot UUID is already present in every Paxos network message: Prepare, Propose, and Commit all carry the same ballot. The instrumentation just records it. So when events arrive from three different nodes, the ballot is the natural correlation key and every LEGACY_PREPARE, LEGACY_PROPOSE, and COMMIT event that belongs to the same round carries the same ballot string. The tool groups by ballot, and then across ballots by `roundId`.

**`roundId`:** This is the one piece of state that does not travel over the wire. It is generated once at the top of `StorageProxy.doPaxos()`, an UUID that identifies one LWT call from the client. If Paxos is retried (because another coordinator bumped the ballot), `doPaxos()` loops and generates a new ballot but keeps the same `roundId`. This is how the tool groups multiple ballot attempts due to contention retries, in-flight recovery (relay-race analogy), etc., under a single *`"Round N"` header*. Replica events have no `roundId`; they are linked to a round by matching their ballot to a coordinator event that does carry the `roundId`.

Currently, the store is a plain in-memory structure with a clean interface. Passing `--latency` to `paxostrace` will capture a per-phase latency breakdown of `prepare`, `check`, `propose`, `commit`, and `total` per ballot attempt, with some cross-node approximations for phases involving multiple nodes.

#### Some remaining directions worth exploring:

- **Contention metrics:** Counting PREPARE_REJECT events per partition over a time window. A partition that keeps getting rejected is a hot key. Right now we can see rejections in the trace; turning them into a metric would let us alert on them.
- **Persistent event log:** The current store is in-memory and evicts. Plugging a background writer that appends events to a structured log file on disk would give us post-incident forensics without having to reproduce the workload.
- **Replica sharpness view:** Because every replica event includes the node address, one could pivot the data: instead of grouping by round, we group by node and see which replicas are consistently slower to promise or accept.

The instrumentation itself, the `PaxosTraceStore.add()` call sites in `PaxosState.java`, becomes the extension point. We can simply add a field to `PaxosTraceEvent`, capture it at the call site, and it flows through JMX to the tool automagically.

## Fín
---

Well that was quite a bit about the internals of LWTs in Apache Cassandra. Hopefully, this makes it pretty clear in terms of the overhead that we inherit to achieve linearizable consistency. The consensus tracetool also gives a real concrete feel to the entire Paxos back-and-forth in order to settle on a single-partition update and the nuances of what could be easily misunderstood.

Will contemplate if anything was missed out in this post w.r.t *LWTs*, and if it would be worth another article in the future.

Until then!
