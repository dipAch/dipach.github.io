---
layout: post
title: Clone a Graph
tags: [graph, algorithms]
---

## Introduction

Having worked close to a decade in software engineering, recently I went back to one of my favorite areas in computer science application - Graphs

Along the way, I came across few interesting problems that highlight some important aspects of graph traversals.

Below is one such problem.

## Clone a Graph

You are given the reference of a node in a connected undirected graph.
Return a deep copy (clone) of the graph.

Each node in the graph contains a value and a list of its neighbors.

There are n nodes in the graph each with a unique value from 0 to n-1.

```
class Node {
    public int value;
    public List<Node> neighbors;
}
```

The given node will have a value of 0. You need to return a copy of this node as a reference to the cloned graph.

### Example
You can add images using markdown syntax:
![Clone Graph Representation](/images/clone-graph.svg)

### Solution
```java
/* This is the Node class definition
class Node {
public
    int value = 0;
    ArrayList<Node> neighbors = new ArrayList<Node>();
    Node(int val) { value = val; }
};
*/

class Solution {
	
    Node cloneGraph(Node node) {
        // add your code here
		Queue<Node> queue = new LinkedList<>();
		Map<Node, Node> originalToClone = new HashMap<>();
		queue.offer(node);
		
		while (!queue.isEmpty()) {
			Node currNode = queue.poll();
			originalToClone.putIfAbsent(currNode, new Node(currNode.value));
			Node clonedNode = originalToClone.get(currNode);
			
			for (Node neighbor : currNode.neighbors) {
				Node clonedNeighbor = originalToClone.putIfAbsent(
					neighbor, new Node(neighbor.value));
				
				if (clonedNeighbor == null) {
					clonedNeighbor = originalToClone.get(neighbor);
					queue.offer(neighbor);
				}
				
				clonedNode.neighbors.add(clonedNeighbor);
			}
		}
		
		return originalToClone.get(node);
    }
}
```

### Runtime Complexity
- Time: O(V + 2E)
- Space: O(V)

## Conclusion
Wrap up your post with a conclusion that summarizes the key points.
