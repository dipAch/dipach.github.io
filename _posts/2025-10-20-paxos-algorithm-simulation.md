---
layout: post
title: "single decree paxos simulation -- let's agree to disagree"
tags: [distributed-systems, consensus, simulation]
---

In one of my previous post, I laid down the foundational concepts that make an algorithm like Paxos tick. In this post, I want to provide a more practical simulation based feel of the concepts. It tries as best as possible to capture the core principles and ideas behind this important and interesting alogirthm.

Whether you **agree or disagree**, I'll leave the best to you 😜

## What is Paxos?
---

Being an avid follower of DRY principle, I would encourage the reader to refer to my previous post on the topic @ **[Paxos Primer](https://dipach.github.io/understanding-paxos-algorithm/)**

## Simulation Components
---

We create a simulation cluster class that orchestrates the various phases of the algorithm among the participants. I'll include the various code blocks progressively as needed, to avoid cognitive clutter.

```java
public class Cluster {

    private final List<Node> acceptors = new ArrayList<>();

    private final List<Node> learners = new ArrayList<>();

    private int totalNodes = 0;

    private final Map<Integer, Node> participatingClusterNodes = new HashMap<>();

    ...

    public Cluster(int participatingNodes) {
        for (int i = 1; i <= participatingNodes; i++) {
            participatingClusterNodes.put(i, new Node(i));
            totalNodes++;
        }

        acceptors.addAll(participatingClusterNodes.values());
        learners.addAll(List.of(new Node(++participatingNodes), new Node(++participatingNodes)));
    }

    public void simulateConsensus(int rounds) {
        ...
        ...
    }

    private void commitPhase(int proposedValue, long proposalNumber, Node proposerNode) {
        ...
        ...
    }

    public static void main(String[] args) {
        Cluster cluster = new Cluster(5);
        ...
        ...
    }
```

Here the cluster takes in the number of participating nodes in the actual decision making process. This excludes nodes labeled as learners.

To initialize the cluster, we pass the participating nodes as an argument. The participating nodes form the acceptor set and also take up the role of a proposer (chosen randomly). We also initialize a convenience map for lookup by node-id.

The learners are additional nodes that don't participate in the voting process nor do they propose any values to the cluster. They just observe and learn whatever values are agreed upon by the majority of participating nodes.

So far so good.

### **PREPARE:**
---

Firstly, Paxos solves the problem of unanimously arriving at a decision, by a majority voting play.

So in-order to make a decision, we need to have a topic that requires agreeing upon. This topic is nothing but for our purposes of the simulation, a value that will be proposed by some participating node(s). We call such a node, **"Proposer"**.

There could be multiple concurrent proposers, each wanting their value to be finalized by the cluster participants.

Let's take a closer look at the code.

```java
public void simulateConsensus(int rounds) {
        int randomInt = random.nextInt(totalNodes) + 1;
        System.out.println("[Random proposer NodeId]=" + randomInt);
        Node proposerNode = participatingClusterNodes.get(randomInt);
        ...
        ...
```

In the above code block, a random node is chosen out of the participants to act as a proposer node. Once we decide on the node-id, we fetch the node object from our convenience map using the node-id.

To the **`simulateConsensus`** method above, we pass the number of decision making rounds we want to simulate. For simulating a single-decree paxos it would be **1**.

In our sample simulation run, we are choosing **2** proposers trying to propose their chosen values simultaneously. This will provide a good feel for the contention experienced in deciding a single value from a set of proposers in a single round.

Once a proposer node is selected, it chooses a value to be proposed to all the other participating nodes. It will also choose a **proposal number** to tag its proposed value.

Below code delves into the **prepare phase's** algorithm,

```java
for (int r = 1; r <= rounds; r++) {
    long n;
    int value = random.nextInt(1000) + 1; // 1 - 1000
    List<Accept> prepareResponses = new ArrayList<>();
    System.out.println();
    System.out.println("> [Proposer Node]=" + proposerNode);
    System.out.println();
    System.out.println("> [Proposer Node]=" + proposerNode + " ========== [Init Round " + r + "] ==========");
    System.out.println();
    System.out.println("> [Proposer Node]=" + proposerNode + " - Entering prepare phase");

    while (true) {
        n = proposerNode.getNextInt();
        System.out.println("> [Proposer Node]=" + proposerNode + " - [Proposal Number]=" + n);
        AtomicInteger prepareCount = new AtomicInteger(0);
        final long candidateProposalNumber = n;
        acceptors.forEach(node -> {
            Accept response = node.prepare(candidateProposalNumber);
            System.out.println("> [Proposer Node]=" + proposerNode + " -- Node=" + node + ", Response=" + response);

            if (response != null) {
                prepareCount.getAndIncrement();
                prepareResponses.add(response);
            }
        });

        if (prepareCount.get() >= (totalNodes / 2) + 1) {
            System.out.println();
            System.out.println("> [Proposer Node]=" + proposerNode + " - Achieved prepare quorum");
            break;
        }
    }
```

The next step is to send the proposal number to all the connected nodes and try to get a confirmation/promise that the proposal number is valid and the acceptors are ready to move to the next step, which is accepting/deciding on the proposed value. This step means that the acceptors promise to reject any proposal number lesser than the promised proposal number.

This also implies having a monotonically increasing cluster wide unique proposal number for correctness.

The action of sending the proposal number to connected nodes is usually called the **PREPARE Phase**.

Below is the **PROMISE** logic at each node for the **PREPARE Phase**,

```java
public Accept prepare(long n) {
    if (n > promisedN) {
        promisedN = n;

        return new Accept(acceptedN, acceptedValue);
    }

    return null;
}
```

We wait for a majority of responses to proceed further. Once the responses come in and we obtain a majority (or quorum), we can safely say that the proposal has achieved **PREPARE** quorum. In our above simulation code (which follows a synchronous flow of execution), we will receive it from all the participating nodes without a hitch, but in a real world scenario we only wait for a majority (owing to network delays, processing delays and what not).

Once we have the proposal number promised by a majority, we need to double confirm on the proposal value. This is where if a value was already decided or agreed upon by the cluster previously, it will prevail and only that same accepted value can be re-proposed. The new value will be overriden in such a case. This creates a safety mechanism that a decided value is not overridden by a new value.

Imagine if I decided/confirmed on a value for **index number 2** on my log (think an array data-structure). And later that value is changed to something else. That would be problematic. So, once confirmed, the value cannot change.

Here, we try to arrive at a decision and post that all new decisions (possibly with newer and different values), will always be **overriden** by the initially decided/accepted value.

So, the decided value ~~will~~ **can** never change. Think of it as the decision made for a single slot in a state-machine log having multiple indices (like an array). Once the slot value in the log is finalized, any contention for that slot at any later time should always preserve the value that was initially decided upon. No change can occur. Once committed and promised to the client, it is done for good! Amen!

Below code takes care of overriding any new values with a previously decided and finalized value (already reached consensus, that is).

```java
ToLongFunction<Accept> getValue = Accept::acceptedN;
Comparator<Accept> comparingByValue = Comparator.comparingLong(getValue);
Optional<Accept> maxTuple = prepareResponses.stream().max(comparingByValue);

if (maxTuple.isPresent()) {
    value = maxTuple.get().acceptedValue() == -1 ? value : maxTuple.get().acceptedValue();
}
```

Here, we try to get the accepted proposal number with the highest value from the set of responses. For the highest accepted proposal number, if we have an associated accepted value, we will re-propose that same value instead of a new one. Else, we will propose with a new value.

A proposal with a lower number will be rejected if the participating nodes (aka acceptors) have already promised on a higher proposal number. **Acceptors** promise on a proposal number iff that is higher than any other promised proposal number.

Proposing a new value will happen only for the first time a decision is being made. A correctly implemented algorithm, will always decide on the same accepted value for all subsequent proposals.

So, in a nutshell if a proposer node is delayed in sending a proposal number which becomes outdated, it will get rejected.

Similarly, if a new higher numbered proposal is promised by a majority in the meantime post a prepare quorum win, the older promised proposal number will be dis-regarded and its associated proposal value will not be comsidered for acceptance. Basically **ACCEPT Phase** with the outdated proposal number is outright rejected.

So, a proposal number can keep on increasing motonically, but if a value is already decided it will remain the same each time for every monotonically increasing proposal number.

Still there with me?

### **ACCEPT:**
---

**Acceptors** have 2 roles to play, largely:
- In the **PREPARE Phase**, they promise on a proposal number.
- In the **ACCEPT Phase**, they need to reach consensus on a proposed value for the already promised proposal number.

Let's now go over the code that simulates this phase.

```java
final int proposedValue = value;
final long proposalNumber = n;
AtomicInteger acceptCount = new AtomicInteger(0);

System.out.println();
System.out.println("> [Proposer Node]=" + proposerNode + " - Entering accept phase");
System.out.println("> [Proposer Node]=" + proposerNode + " - [Proposal Number]=" + proposalNumber + ", [Proposed Value]=" + proposedValue);

acceptors.forEach(node -> {
    System.out.println("> [Proposer Node]=" + proposerNode + " ** [Before] Node=" + node);
    Accept response = node.accept(proposedValue, proposalNumber);
    System.out.println("> [Proposer Node]=" + proposerNode + " -- [After] Node=" + node + ", Response=" + response);

    if (response != null) {
        acceptCount.getAndIncrement();
    }
});

if (acceptCount.get() >= (totalNodes / 2) + 1) {
    // Achieved accept quorum and initiate learning/commit phase.
} else {
    // Could not reach accept quorum.
}
```

In the above code block, once we have achieved a **PREPARE** quorum and locked in on the value to propose, we send out the accept messages to the participating nodes.

Below is the **ACCEPT** logic at each node,

```java
public Accept accept(int value, long n) {
    if (n >= promisedN) {
        promisedN = n;
        acceptedN = n;
        acceptedValue = value;

        return new Accept(acceptedN, acceptedValue);
    }

    return null;
}
```

If we receive a proposal value for a **currently** promised proposal number, we go ahead and accept/finalize that proposal value. But, if the proposal number accompanying the accept request is outdated, we will reject the request.

Once the responses for the accept requests are received and depending on quorum conditions, a proposed value is either agreed upon or rejected.

And with that we can say that the cluster has reached a consensus for a particular round. In Multi-Paxos, this could mean deciding the value for a particular slot.

### **Learners:**
---

Learners are the outcome consumers for a decision. Once a decision is majority accepted, the same is propagated to the learners set.

This is also called as the commit phase, as we are saying that this decision will not be reversed going forward.

```java
public void simulateConsensus(int rounds) {
    ...
    ...

    if (acceptCount.get() >= (totalNodes / 2) + 1) { // Once majority nodes have accepted the value.
        ...
        System.out.println("> [Proposer Node]=" + proposerNode + " - Starting learning phase");
        commitPhase(proposedValue, proposalNumber, proposerNode); // Call to propagate the decision
        ...
    }
    ...
}

private void commitPhase(int proposedValue, long proposalNumber, Node proposerNode) {
    learners.forEach(node -> {
        System.out.println("> [Proposer Node]=" + proposerNode + " -- [Before] - [Learner Node]=" + node);
        node.learn(proposerNode.getPromisedN(), proposedValue, proposalNumber);
        System.out.println("> [Proposer Node]=" + proposerNode + " -- [After] - [Learner Node]=" + node);
    });
}
```

And with this a paxos cluster arrives at a decision selecting a particular proposed value from a set of possible values. In single-decree variant, no new value can be proposed once the decision is committed.

We can internalize this by trying to start a new round of decision making. We will see that the value that was accepted/committed in the previous round, is only re-proposed by the new proposal number.

## How the Simulation Works?
---

So in our simulation run, I am going to do two things:
- First is to get the cluster decide on a value for the first time. In this decision making round, I'll create two contending proposers trying to get their proposal accepted by a majority.
- Next, I'll try to get a new proposer to initiate a new proposal. This will be done after the initial round's outcome is committed. So, in this case the proposal will end up proposing the accepted value again from the previous round.

And with this we would have showcased consistentency and durability in making decisions in a multi-node cluster setup (with a majority nodes up and connected).

### Example execution (with commentary):
---

> **STEP 1.1**

Firstly we choose proposers for the first round. The below program output captures that,

```
===== Choose a new value =====
[Random proposer NodeId]=4
[Random proposer NodeId]=5


> [Proposer Node]=Node{nodeId=5, promisedN=-1, acceptedN=-1, acceptedValue=-1}

> [Proposer Node]=Node{nodeId=4, promisedN=-1, acceptedN=-1, acceptedValue=-1}
```

> **STEP 1.2**

With our proposers chosen at random, now we will step-in to the **PREPARE Phase**. Both nodes try to parallely generate a proposal number to label their proposals.

To recap, our proposers are:
- **Node-4** and,
- **Node-5**

```
> [Proposer Node]=Node{nodeId=4, promisedN=-1, acceptedN=-1, acceptedValue=-1} ========== [Init Round 1] ==========

> [Proposer Node]=Node{nodeId=5, promisedN=-1, acceptedN=-1, acceptedValue=-1} ========== [Init Round 1] ==========

> [Proposer Node]=Node{nodeId=5, promisedN=-1, acceptedN=-1, acceptedValue=-1} - Entering prepare phase
> [Proposer Node]=Node{nodeId=4, promisedN=-1, acceptedN=-1, acceptedValue=-1} - Entering prepare phase
> [Proposer Node]=Node{nodeId=4, promisedN=-1, acceptedN=-1, acceptedValue=-1} - [Proposal Number]=2
> [Proposer Node]=Node{nodeId=5, promisedN=-1, acceptedN=-1, acceptedValue=-1} - [Proposal Number]=1
```

> **STEP 1.3**

Once the proposal number is chosen as seen above, the nodes will send out prepare requests to all it connected nodes. Both the proposers will wait for the responses from the other acceptor nodes for their respective proposals.

Below we can see that both the proposer nodes achieve majority proposal promise by the connected acceptor nodes. This provides a good view as to that two or more proposers can get their proposal numbers approved when contending in parallel.

This strictly requires having a unique proposal number for each proposer (for correctness), and the one with the highest proposal number will actually be able to win and get the **proposal** accepted.

```
> [Proposer Node]=Node{nodeId=5, promisedN=-1, acceptedN=-1, acceptedValue=-1} -- Node=Node{nodeId=1, promisedN=1, acceptedN=-1, acceptedValue=-1}, Response=Accept[acceptedN=-1, acceptedValue=-1]
> [Proposer Node]=Node{nodeId=4, promisedN=-1, acceptedN=-1, acceptedValue=-1} -- Node=Node{nodeId=1, promisedN=1, acceptedN=-1, acceptedValue=-1}, Response=Accept[acceptedN=-1, acceptedValue=-1]
> [Proposer Node]=Node{nodeId=5, promisedN=-1, acceptedN=-1, acceptedValue=-1} -- Node=Node{nodeId=2, promisedN=1, acceptedN=-1, acceptedValue=-1}, Response=Accept[acceptedN=-1, acceptedValue=-1]
> [Proposer Node]=Node{nodeId=4, promisedN=-1, acceptedN=-1, acceptedValue=-1} -- Node=Node{nodeId=2, promisedN=2, acceptedN=-1, acceptedValue=-1}, Response=Accept[acceptedN=-1, acceptedValue=-1]
> [Proposer Node]=Node{nodeId=5, promisedN=-1, acceptedN=-1, acceptedValue=-1} -- Node=Node{nodeId=3, promisedN=1, acceptedN=-1, acceptedValue=-1}, Response=Accept[acceptedN=-1, acceptedValue=-1]
> [Proposer Node]=Node{nodeId=4, promisedN=-1, acceptedN=-1, acceptedValue=-1} -- Node=Node{nodeId=3, promisedN=2, acceptedN=-1, acceptedValue=-1}, Response=Accept[acceptedN=-1, acceptedValue=-1]
> [Proposer Node]=Node{nodeId=5, promisedN=-1, acceptedN=-1, acceptedValue=-1} -- Node=Node{nodeId=4, promisedN=1, acceptedN=-1, acceptedValue=-1}, Response=Accept[acceptedN=-1, acceptedValue=-1]
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1} -- Node=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1}, Response=Accept[acceptedN=-1, acceptedValue=-1]
> [Proposer Node]=Node{nodeId=5, promisedN=1, acceptedN=-1, acceptedValue=-1} -- Node=Node{nodeId=5, promisedN=1, acceptedN=-1, acceptedValue=-1}, Response=Accept[acceptedN=-1, acceptedValue=-1]
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1} -- Node=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1}, Response=Accept[acceptedN=-1, acceptedValue=-1]


> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} - Achieved prepare quorum
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1} - Achieved prepare quorum
```

> **STEP 2.1**

So once both proposer nodes have achieved prepare quorum, they will try to put forward their proposed values for the majority to accept.

This marks the beginning of the **ACCEPT Phase**. We can see below,
- **Node-4** with **proposal number: 2** and **proposed value: 936**
- **Node-5** with **proposal number: 1** and **proposed value: 416**

So, every proposer sends the accept request to all of its connected nodes and waits for a majority response.

We see that the accept request with the lower proposal number, even though it was promised previously wil not have it proposed value accepted. All nodes accpet the proposed value of the proposal number with the highest value which was promised most recently.

This provides safety that older proposals don't come at a later time and change the most recent proposal's data value.

Below let's shift our attention to **Node-1**'s response to the parallel accept requests,
- We see that it had promised to accept proposals with proposal number **1** or more. **Node-5**'s proposal number is 1.
- Now when **Node-4** sends its accept request, the lagging node will update the promise to **Node-4**'s proposal number, i.e., **2** and it will reject **Node-5**'s accept request which had a lower proposal number.

Hence, Node-5 gets a **null** response whereas Node-4 got a valid accept response, **Response=Accept[acceptedN=2, acceptedValue=936]**

Finally, we can see above that **Node-4** achieved **accept quorum** and **Node-5** failed in getting a majority vote for its proposal.

```
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} - Entering accept phase
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1} - Entering accept phase
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} - [Proposal Number]=1, [Proposed Value]=416
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1} - [Proposal Number]=2, [Proposed Value]=936
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1} ** [Before] Node=Node{nodeId=1, promisedN=1, acceptedN=-1, acceptedValue=-1}
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} ** [Before] Node=Node{nodeId=1, promisedN=1, acceptedN=-1, acceptedValue=-1}
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} -- [After] Node=Node{nodeId=1, promisedN=2, acceptedN=2, acceptedValue=936}, Response=null
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1} -- [After] Node=Node{nodeId=1, promisedN=2, acceptedN=2, acceptedValue=936}, Response=Accept[acceptedN=2, acceptedValue=936]
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} ** [Before] Node=Node{nodeId=2, promisedN=2, acceptedN=-1, acceptedValue=-1}
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} -- [After] Node=Node{nodeId=2, promisedN=2, acceptedN=-1, acceptedValue=-1}, Response=null
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1} ** [Before] Node=Node{nodeId=2, promisedN=2, acceptedN=-1, acceptedValue=-1}
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} ** [Before] Node=Node{nodeId=3, promisedN=2, acceptedN=-1, acceptedValue=-1}
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} -- [After] Node=Node{nodeId=3, promisedN=2, acceptedN=-1, acceptedValue=-1}, Response=null
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1} -- [After] Node=Node{nodeId=2, promisedN=2, acceptedN=2, acceptedValue=936}, Response=Accept[acceptedN=2, acceptedValue=936]
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} ** [Before] Node=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1}
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1} ** [Before] Node=Node{nodeId=3, promisedN=2, acceptedN=-1, acceptedValue=-1}
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} -- [After] Node=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1}, Response=null
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} ** [Before] Node=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1}
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} -- [After] Node=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1}, Response=null

> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1} -- [After] Node=Node{nodeId=3, promisedN=2, acceptedN=2, acceptedValue=936}, Response=Accept[acceptedN=2, acceptedValue=936]
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1} ** [Before] Node=Node{nodeId=4, promisedN=2, acceptedN=-1, acceptedValue=-1}
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} -- [After] Node=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936}, Response=Accept[acceptedN=2, acceptedValue=936]
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} ** [Before] Node=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1}
> [Proposer Node]=Node{nodeId=5, promisedN=2, acceptedN=-1, acceptedValue=-1} ***** [Round 1] ***** Could not reach quorum
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} -- [After] Node=Node{nodeId=5, promisedN=2, acceptedN=2, acceptedValue=936}, Response=Accept[acceptedN=2, acceptedValue=936]

> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} ----- [Status Round 1] -----

> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} - Achieved accept quorum
```

> **STEP 2.2**

Now that a value has been decided in consortium, we will propagate that decision to the learners group in order to commit the process. below we can see the call made to all the learners with the accepted data valie.

```
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} - Initial State :: [Learner Nodes]=[Node{nodeId=6, promisedN=-1, acceptedN=-1, acceptedValue=-1}, Node{nodeId=7, promisedN=-1, acceptedN=-1, acceptedValue=-1}]

> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} - Starting learning phase
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} -- [Before] - [Learner Node]=Node{nodeId=6, promisedN=-1, acceptedN=-1, acceptedValue=-1}
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} -- [After] - [Learner Node]=Node{nodeId=6, promisedN=2, acceptedN=2, acceptedValue=936}
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} -- [Before] - [Learner Node]=Node{nodeId=7, promisedN=-1, acceptedN=-1, acceptedValue=-1}
> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} -- [After] - [Learner Node]=Node{nodeId=7, promisedN=2, acceptedN=2, acceptedValue=936}

> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} - Final State :: [Learner Nodes]=[Node{nodeId=6, promisedN=2, acceptedN=2, acceptedValue=936}, Node{nodeId=7, promisedN=2, acceptedN=2, acceptedValue=936}]

> [Proposer Node]=Node{nodeId=4, promisedN=2, acceptedN=2, acceptedValue=936} ***** [End of Round 1] *****
```

> **New Independent Round**

After the first round's outcome is committed in hostory, we will test the durability of the decision. Will will initiate a new round that tries to challenge the previous value. But we can see that after much hula-hoop, it will override its new value with the previously accepted data value (or the learned value).

```
===== A new round after commit =====
[Random proposer NodeId]=2

> [Proposer Node]=Node{nodeId=2, promisedN=2, acceptedN=2, acceptedValue=936}

> [Proposer Node]=Node{nodeId=2, promisedN=2, acceptedN=2, acceptedValue=936} ========== [Init Round 1] ==========

> [Proposer Node]=Node{nodeId=2, promisedN=2, acceptedN=2, acceptedValue=936} - Entering prepare phase
> [Proposer Node]=Node{nodeId=2, promisedN=2, acceptedN=2, acceptedValue=936} - [Proposal Number]=3
> [Proposer Node]=Node{nodeId=2, promisedN=2, acceptedN=2, acceptedValue=936} -- Node=Node{nodeId=1, promisedN=3, acceptedN=2, acceptedValue=936}, Response=Accept[acceptedN=2, acceptedValue=936]
> [Proposer Node]=Node{nodeId=2, promisedN=3, acceptedN=2, acceptedValue=936} -- Node=Node{nodeId=2, promisedN=3, acceptedN=2, acceptedValue=936}, Response=Accept[acceptedN=2, acceptedValue=936]
> [Proposer Node]=Node{nodeId=2, promisedN=3, acceptedN=2, acceptedValue=936} -- Node=Node{nodeId=3, promisedN=3, acceptedN=2, acceptedValue=936}, Response=Accept[acceptedN=2, acceptedValue=936]
...
...

> [Proposer Node]=Node{nodeId=2, promisedN=3, acceptedN=2, acceptedValue=936} - Achieved prepare quorum

> [Proposer Node]=Node{nodeId=2, promisedN=3, acceptedN=2, acceptedValue=936} - Entering accept phase
> [Proposer Node]=Node{nodeId=2, promisedN=3, acceptedN=2, acceptedValue=936} - [Proposal Number]=3, [Proposed Value]=936
...
...

> [Proposer Node]=Node{nodeId=2, promisedN=3, acceptedN=3, acceptedValue=936} - Achieved accept quorum

> [Proposer Node]=Node{nodeId=2, promisedN=3, acceptedN=3, acceptedValue=936} - Initial State :: [Learner Nodes]=[Node{nodeId=6, promisedN=2, acceptedN=2, acceptedValue=936}, Node{nodeId=7, promisedN=2, acceptedN=2, acceptedValue=936}]

> [Proposer Node]=Node{nodeId=2, promisedN=3, acceptedN=3, acceptedValue=936} - Starting learning phase
...
...

> [Proposer Node]=Node{nodeId=2, promisedN=3, acceptedN=3, acceptedValue=936} - Final State :: [Learner Nodes]=[Node{nodeId=6, promisedN=3, acceptedN=3, acceptedValue=936}, Node{nodeId=7, promisedN=3, acceptedN=3, acceptedValue=936}]

> [Proposer Node]=Node{nodeId=2, promisedN=3, acceptedN=3, acceptedValue=936} ***** [End of Round 1] *****
```

See the monotonic increase in the **promisedN** and **acceptedN** values above, while the **acceptedValue** remains the same.

## The Code
---

The code for the simulation program can be found @ **[Paxos Simulation](https://github.com/dipAch/paxos-simulation)**

## Conclusion
---

The simulation demonstrates that Paxos achieves safety through two key mechanisms: monotonically increasing proposal numbers prevent stale proposals from corrupting the system, and the prepare phase's obligation to return previously accepted values ensures that committed decisions are immutable.

Even when multiple proposers compete simultaneously, the highest proposal number wins—yet the accepted value, once committed, never changes regardless of how many new rounds are initiated.

What makes this algorithm particularly interesting is its ability to handle failures and contention without coordination. Each node follows local rules, yet the collective behavior guarantees consensus. The learners simply observe the outcome without participating in the voting, making the system scalable for read-heavy workloads.

Play around and possibly extend the simulation parameters — vary the number of nodes, introduce failures, or increase concurrent proposers.

Happy learning!
