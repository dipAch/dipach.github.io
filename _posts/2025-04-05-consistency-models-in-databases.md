---
layout: post
title: understanding serializability, strict serializability, and linearizability
tags: [databases, concurrency, distributed-systems]
---

In the world of databases and distributed systems, ensuring correct behavior when multiple operations happen concurrently is crucial. This post explores three fundamental concepts that help us understand and guarantee correct behavior in concurrent systems: **Serializability**, **Strict Serializability**, and **Linearizability**. We'll break down each concept with practical examples and explain how they differ from each other.

Linearizability and serializability are both important properties about interleavings of operations in databases and distributed systems. In database context, interleavings are the ordering of operations that can happen in concurrent transactional systems.

## Serializability

Serializability is a property that ensures that the concurrent execution of transactions produces the same result as some serial execution of those transactions. 

Serializability is a guarantee about transactions, or groups of one or more operations over one or more objects. It guarantees that the execution of a set of transactions (usually containing read and write operations) over multiple items is equivalent to some serial execution (total ordering) of the transactions.

In other words, even though operations are happening concurrently, the final state of the database should be equivalent to some sequential ordering of those namespaced/grouped operations (i.e., transactions).

Serializability is the traditional “I,” or isolation, in ACID. If users’ transactions each preserve application correctness (“C,” or consistency, in ACID), a serializable execution also preserves correctness. Therefore, serializability is a mechanism for guaranteeing database correctness.

Unlike linearizability, serializability does not impose any real-time constraints on the ordering of transactions. Serializability does not imply any kind of deterministic order—it simply requires that some equivalent serial execution exists.

### Example: Bank Account Transfers

Consider two bank accounts, A and B, each starting with $100. Two transactions happen concurrently:

```
Transaction T1: Transfer $50 from A to B
Transaction T2: Transfer $30 from B to A
```

A serializable execution might look like this:

```
T1: Read A ($100)
T1: Read B ($100)
T1: Write A ($50)
T1: Write B ($150)
T2: Read A ($50)
T2: Read B ($150)
T2: Write A ($80)
T2: Write B ($120)
```

The final state (A=$80, B=$120) is equivalent to executing T1 followed by T2 sequentially. This is a valid serializable execution.

### Key Points about Serializability:
- Only requires that the final state be equivalent to some serial execution
- Doesn't specify the order of operations
- Allows for different valid outcomes based on the chosen serial order

## Strict Serializability

Simply put, combining serializability and linearizability yields strict serializability. But what does that even mean?

Strict Serializability adds an important constraint to basic serializability: the order of operations must respect their real-time ordering. This means that if operation A completes before operation B starts, then A must appear before B in the serial order.

Hence, with strict serializability, transaction behavior is equivalent to some serial execution, and the serial order corresponds to real time.

For example, say we begin and commit transaction T1, which writes to item x, and later begin and commit transaction T2, which reads from x. A database providing strict serializability for these transactions will place T1 before T2 in the serial ordering, and T2 will read T1’s write. A database providing serializability (but not strict serializability) could order T2 before T1.

### Example: Online Shopping System

Consider an online shopping system where two users are trying to buy the last item in stock:

```
Time 1: User A starts checkout
Time 2: User B starts checkout
Time 3: User A completes purchase
Time 4: User B completes purchase
```

In a strictly serializable system:
- User A's purchase must appear before User B's in the serial order
- The system must prevent User B from purchasing the item
- The final state must reflect that User A got the item

### Key Points about Strict Serializability:
- Maintains real-time ordering of operations
- Provides stronger guarantees than basic serializability, in essence upholds real-time ordering for any serial ordering of transactions
- Ensures that operations respect their actual time of occurrence

## Linearizability

Linearizability is a property that ensures that operations appear to take effect instantaneously at some point between their invocation and completion. It's a stronger property than strict serializability and is particularly important in distributed systems.

Linearizability as a consistency model inclines more towards data recency. Once a registry value is written, any following read operations will find the very same value, as long as the registry doesn't undergo any more modifications.

In plain english, under linearizability, writes should appear to be instantaneous. Imprecisely, once a write completes, all later reads (where “later” is defined by wall-clock start time) should return the value of that write or the value of a later write. Once a read returns a particular value, all later reads should return that value or the value of a later write.

Linearizability is what the CAP Theorem calls **"Consistency"**.

A single site data system always exhibits linearizability.

### Instantaneous reads and writes

![Linearizability Representation](/images/linearizability-single-site.png)

### Example: Distributed Counter

Consider a distributed counter that multiple clients can increment:

```
Client 1: Increment counter (starts at 0)
Client 2: Read counter
Client 3: Increment counter
```

In a linearizable system:
- Each increment must appear to happen at a single point in time
- All clients must see the same sequence of values
- The final state must be consistent with the real-time ordering

### Key Points about Linearizability:
- Operations appear to take effect instantaneously
- Provides a single, consistent view of the system
- Stronger than strict serializability

## Comparing the Properties

Let's compare these properties using a simple example of a shared counter:

```
Initial state: counter = 0

Operation A: Increment counter (starts at t1, completes at t2)
Operation B: Read counter (starts at t3, completes at t4)
Operation C: Increment counter (starts at t5, completes at t6)
```

Where t1 < t3 < t5 < t2 < t4 < t6

### Serializable Execution:
- Any order that produces a valid final state
- Could show B reading 0, 1, or 2
- Final state must be counter = 2

### Strictly Serializable Execution:
- Must respect real-time ordering
- B must read either 0 or 1 (since it starts before A completes)
- Final state must be counter = 2

### Linearizable Execution:
- Each operation appears to happen at a single point
- B must read exactly 1 (since it happens between A and C)
- Final state must be counter = 2

## Practical Implications

1. **Performance vs. Consistency**:
   - Linearizability provides the strongest guarantees but can impact performance
   - Serializability offers more flexibility for optimization
   - Strict serializability provides a good balance

2. **Use Cases**:
   - Linearizability: Financial systems, leader election
   - Strict Serializability: Most database transactions
   - Serializability: Systems where final consistency is sufficient

3. **Implementation Considerations**:
   - Linearizability often requires coordination between nodes
   - Serializability can be achieved with local coordination
   - Strict serializability requires both local and global ordering

## Conclusion

Understanding these properties is crucial for designing and implementing distributed systems. While linearizability provides the strongest guarantees, it comes with performance costs. Serializability offers more flexibility but weaker guarantees. Strict serializability strikes a balance between the two, making it suitable for many practical applications.

The choice between these properties depends on your specific requirements:
- How critical is real-time ordering?
- What are your performance requirements?
- What level of consistency do your users expect?

## References

1. Herlihy, M. P., & Wing, J. M. (1990). Linearizability: A correctness condition for concurrent objects
2. Papadimitriou, C. H. (1979). The serializability of concurrent database updates
3. Adya, A. (1999). Weak consistency: A generalized theory and optimistic implementations for distributed transactions 