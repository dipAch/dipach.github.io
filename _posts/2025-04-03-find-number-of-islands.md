---
layout: post
title: Number of Islands
tags: [graph, dfs, algorithms]
---

## Problem Statement

You are given a 2-D matrix surface of size `n x m`. Each cell of the surface is either 1 (land) or 0 (water).
Find the number of islands on the surface.

An island is surrounded by water and is formed by connecting adjacent lands horizontally or vertically. Assume all four edges of the surface are all surrounded by water.

### Example

```
Input: surface = [
  [1, 1, 0, 0, 0],
  [1, 1, 0, 0, 0],
  [0, 0, 1, 0, 0],
  [0, 0, 0, 1, 1]
]

Num islands: 3
```

## Solution

```java
class Solution {
	
    int getNumberOfIslands(int[][] surface) {
        int n = surface.length;
        int m = surface[0].length;
        int[][] visited = new int[n][m];
        int ans = 0;
        
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < m; j++) {
                if (visited[i][j] == 0 && surface[i][j] == 1) {
                    ans++;
                    dfs(surface, visited, i, j);
                }
            }
        }
        
        return ans;
    }

    private void dfs(int[][] surface, int[][] visited, int i, int j) {
        visited[i][j] = 1;
        
        int[] dr = new int[] {-1, 0, 1, 0};
        int[] dc = new int[] {0, 1, 0, -1};
        
        for (int k = 0; k < 4; k++) {
            int nr = i + dr[k];
            int nc = j + dc[k];
            
            if (nr > -1 && nr < surface.length && nc > -1 && nc < surface[0].length
                    && visited[nr][nc] == 0 && surface[nr][nc] == 1) {
                dfs(surface, visited, nr, nc);
            }
        }
    }
}
```

### Solution Explanation

Let's break down how this solution works:

1. **Initialization**:
   - We create a `visited` array of the same size as the input grid to keep track of which cells we've already processed
   - `ans` variable keeps count of the number of islands found

2. **Main Loop**:
   - We iterate through each cell in the grid using nested loops
   - For each cell, we check **two conditions**:
     - `visited[i][j] == 0`: The cell hasn't been visited yet
     - `surface[i][j] == 1`: The cell is land (part of an island)
   - When both conditions are true, we've found a new island:
     - Increment the island counter (`ans++`)
     - Start DFS from this cell to explore the entire island

3. **Depth-First Search (DFS)**:
   - The `dfs` function explores all connected land cells from a given starting point
   - It marks the current cell as visited
   - Uses two arrays `dr` and `dc` to represent the four possible directions:
     - `dr = [-1, 0, 1, 0]` (up, right, down, left)
     - `dc = [0, 1, 0, -1]` (up, right, down, left)
   - For each direction, it checks if:
     - The new position is within grid bounds
     - The cell hasn't been visited
     - The cell is land (value = 1)
   - If all conditions are met, recursively calls DFS on the new cell

4. **Example Walkthrough**:
   ```
   For the input grid:
   [1, 1, 0, 0, 0]
   [1, 1, 0, 0, 0]
   [0, 0, 1, 0, 0]
   [0, 0, 0, 1, 1]

   Step 1: Start at (0,0) - Found land, increment counter to 1
   Step 2: DFS explores all connected 1's in the first island
   Step 3: Continue scanning, find (2,2) - Found new land, increment counter to 2
   Step 4: DFS explores this single-cell island
   Step 5: Continue scanning, find (3,3) - Found new land, increment counter to 3
   Step 6: DFS explores the last island
   ```

### Runtime Complexity
- Time: O(n × m) where n is the number of rows and m is the number of columns
  - Each cell is visited at most once
  - DFS explores each land cell exactly once
- Space: O(n × m)
  - Used by the visited array
  - Recursion stack in worst case (if entire grid is one island)

## Conclusion
We want to go over all possible start points for an island (cells with value 1).
If that start point is not yet visited, that provides us with an opportunity to explore nearby land cells. This also means we have encountered a new land mass.

We can use DFS to explore the nearby land cells and mark them as visited.
