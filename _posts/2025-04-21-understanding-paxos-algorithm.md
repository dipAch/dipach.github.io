---
layout: post
title: "paxos algorithm primer"
tags: [paxos, distributed-consensus]
---

The Paxos algorithm, developed by Leslie Lamport, is one of the most fundamental algorithms in distributed systems. While many articles explain Paxos in theory, few address the practical challenges and nuances that make it both powerful and notoriously difficult to implement correctly. In this post, we'll explore not just how Paxos works, but why it works this way and what challenges you'll face when implementing it.

## Why Paxos Matters

Before diving into the algorithm, let's understand why Paxos is crucial in modern distributed systems:

1. **Database Replication**: When you use a distributed database like MongoDB or Cassandra, Paxos ensures that all replicas stay consistent even when nodes fail or network partitions occur.

2. **Configuration Management**: Systems like etcd and Consul use Paxos variants to maintain consistent configuration across clusters.

3. **Leader Election**: Many distributed systems need a single leader to coordinate operations. Paxos provides a reliable way to elect and maintain leadership.

## The Consensus Problem: More Than Just Agreement

The consensus problem seems simple: get multiple processes to agree on a value. However, the real challenge lies in the constraints:

1. **Safety**: The system must never reach an inconsistent state, even if:
   - Messages are lost
   - Nodes fail
   - Network partitions occur
   - Messages are delayed
   - Messages are duplicated

2. **Liveness**: The system must eventually make progress, even if:
   - New nodes join
   - Failed nodes recover
   - Network partitions heal

## The Basic Paxos Algorithm: Why Three Phases?

Paxos uses three phases not by choice, but by necessity. Each phase serves a specific purpose in maintaining safety and liveness:

### 1. Prepare Phase: The Safety Net

```java
public record Prepare(Ballot ballotNum) {}
public record Promise(Ballot ballotNum, List<Accepted> acceptedProposals) {}
```

The prepare phase isn't just about proposing values—it's about establishing a safety boundary. Here's why:

1. **Ballot Numbers**: They're not just sequence numbers. They establish a total ordering of proposals, crucial for:
   - Resolving conflicts
   - Handling concurrent proposals
   - Ensuring consistency across partitions

2. **Promises**: When an acceptor makes a promise, it's not just agreeing to consider a proposal. It's:
   - Rejecting all proposals with lower ballot numbers
   - Committing to only accept proposals with higher ballot numbers
   - Providing information about previously accepted values

### 2. Accept Phase: The Commitment

```java
public record Accept(int slot, Ballot ballotNum, Proposal proposal) {}
public record Accepted(int slot, Ballot ballotNum) {}
```

The accept phase is where the real magic happens:

1. **Value Selection**: The proposer must choose a value carefully:
   - If no previous proposals exist, it can propose any value
   - If previous proposals exist, it must propose the value with the highest ballot number
   - This rule is crucial for maintaining consistency

2. **Majority Requirement**: Why a majority? Because:
   - It ensures that any two majorities overlap
   - This overlap guarantees that at least one acceptor knows about previous decisions
   - It prevents split-brain scenarios

### 3. Learn Phase: The Propagation

```java
public record Decision(int slot, Proposal proposal) {}
```

The learn phase is often overlooked but is crucial for:
- Ensuring all nodes eventually learn the decision
- Handling node failures and recoveries
- Maintaining consistency across the system

## Real-World Challenges and Solutions

### 1. Performance Optimization

The basic Paxos algorithm is correct but inefficient. Here's how real systems optimize it:

1. **Multi-Paxos**:
   - Instead of running full Paxos for each value
   - Elect a stable leader
   - Skip prepare phase for subsequent proposals
   - Reduces message complexity from O(n²) to O(n)

2. **Batching**:
   - Combine multiple proposals into a single round
   - Reduces network overhead
   - Increases throughput

3. **Pipelining**:
   - Don't wait for one proposal to complete before starting the next
   - Maintains multiple proposals in flight
   - Improves latency

### 2. Handling Real-World Failures

```java
public record Preempted(int slot, Ballot preemptedBy) {}
public record Adopted(Ballot ballotNum, List<Accepted> acceptedProposals) {}
```

Real systems face more than just node failures:

1. **Network Partitions**:
   - Can cause split-brain scenarios
   - Solution: Use timeouts and heartbeat mechanisms
   - Implement proper partition detection

2. **Clock Drift**:
   - Affects timeout calculations
   - Solution: Use logical clocks (Lamport timestamps)
   - Implement clock synchronization

3. **Resource Exhaustion**:
   - Memory leaks from accumulated promises
   - Solution: Implement promise cleanup
   - Use proper resource management

### 3. Implementation Gotchas

1. **Message Ordering**:
   - TCP doesn't guarantee message ordering across connections
   - Solution: Use sequence numbers
   - Implement proper message buffering

2. **State Management**:
   - Need to persist state for recovery
   - Solution: Use write-ahead logging
   - Implement proper checkpointing

3. **Concurrency Control**:
   - Multiple threads accessing shared state
   - Solution: Use proper synchronization
   - Implement thread-safe data structures

## Real-World Example: Distributed Database

Let's see how these concepts apply in practice:

```java
// Client request
public record Invoke(String caller, String clientId, Object inputValue) {}

// Server response
public record Invoked(String clientId, Object output) {}
```

1. **Client Request Handling**:
   - Client sends request to any node
   - Node becomes proposer
   - Generates unique ballot number
   - Starts prepare phase

2. **Leader Election**:
   - Uses Paxos to elect leader
   - Leader maintains stable leadership
   - Handles leader failures

3. **Value Agreement**:
   - Leader proposes value
   - Acceptors vote
   - Majority agreement reached
   - Value committed

4. **Response Handling**:
   - Client receives response
   - System maintains consistency
   - Handles retries and timeouts

## Common Pitfalls and Their Solutions

1. **Split Votes**:
   - Problem: Multiple proposers causing conflicts
   - Solution: Use unique ballot numbers and backoff
   - Implementation: Use node ID + timestamp for ballot numbers

2. **Livelock**:
   - Problem: Continuous proposal conflicts
   - Solution: Implement exponential backoff
   - Implementation: Use randomized timeouts

3. **Performance Issues**:
   - Problem: Too many messages
   - Solution: Use Multi-Paxos optimization
   - Implementation: Maintain stable leadership

## Best Practices from Production Systems

1. **Implementation**:
   - Use proper timeouts (typically 100-500ms)
   - Implement retry mechanisms with exponential backoff
   - Handle all failure cases explicitly
   - Maintain proper logging and metrics

2. **Configuration**:
   - Choose appropriate timeouts based on network characteristics
   - Configure proper majority size (typically 2f + 1 for f failures)
   - Set up monitoring and alerting

3. **Testing**:
   - Test failure scenarios systematically
   - Verify consistency under various conditions
   - Check performance under load
   - Use chaos testing tools

## Conclusion

Paxos is more than just an algorithm—it's a framework for thinking about distributed consensus. The key to successful implementation lies in:

1. Understanding the underlying principles
2. Implementing proper failure handling
3. Using appropriate optimizations
4. Testing thoroughly

Remember that Paxos is not just an algorithm but a family of protocols. Choose the right variant based on your specific needs and constraints.

## References

1. Lamport, Leslie. "Paxos Made Simple." ACM SIGACT News 32, no. 4 (2001): 51-58.
2. Lamport, Leslie. "The Part-Time Parliament." ACM Transactions on Computer Systems 16, no. 2 (1998): 133-169.
3. Chandra, Tushar, et al. "Paxos Made Live: An Engineering Perspective." PODC 2007.
4. Ongaro, Diego, and John Ousterhout. "In Search of an Understandable Consensus Algorithm." USENIX ATC 2014.
5. Burrows, Mike. "The Chubby Lock Service for Loosely-Coupled Distributed Systems." OSDI 2006.

--- 