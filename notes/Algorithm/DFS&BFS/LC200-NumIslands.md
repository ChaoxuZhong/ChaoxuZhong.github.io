# 岛屿数量
## 题目描述

给你一个由 `'1'`（陆地）和 `'0'`（水）组成的二维网格，请你计算网格中岛屿的数量。岛屿总是被水包围，并且每座岛屿只能由水平方向和/或竖直方向上相邻的陆地连接形成。

## 解题思路

1. **DFS策略**：
    - 遍历整个二维网格
    - 每当遇到一个「1」，就以该位置为起点进行DFS标记
    - 在DFS过程中将所有相连的「1」标记为已访问
    - 每完成一次完整的DFS，就代表找到了一个新的岛屿

2. **关键点**：
    - 使用visited数组避免重复访问
    - 定义四个方向的移动数组
    - 处理边界条件
    - 标记已访问的陆地

## 代码实现

```java
class Solution {
    // 定义四个方向：上、下、左、右
    // Define four directions: up, down, left, right
    private int[][] directions = {{1,0},{-1,0},{0,1},{0,-1}};
    private boolean[][] visited;
    private int rowLen;
    private int colLen;
    
    public int numIslands(char[][] grid) {
        // 初始化成员变量
        // Initialize member variables
        rowLen = grid.length;
        colLen = grid[0].length;
        visited = new boolean[rowLen][colLen];
        int res = 0;
        
        // 遍历整个网格
        // Traverse the entire grid
        for(int row = 0; row < rowLen; row++){
            for(int col = 0; col < colLen; col++){
                res += dfs(grid, row, col);
            }
        }
        return res;
    }
    
    private int dfs(char[][] grid, int row, int col){
        // 越界、已访问或是水域，直接返回0
        // Out of bounds, visited, or water area, return 0 directly
        if(row >= rowLen || row < 0 || 
           col >= colLen || col < 0 || 
           visited[row][col] || 
           grid[row][col] != '1'){
            return 0;
        }
        
        // 标记当前位置为已访问
        // Mark current position as visited
        visited[row][col] = true;
        
        // 对四个方向进行DFS搜索
        // Perform DFS search in four directions
        for(int[] direction : directions){
            dfs(grid, row + direction[0], col + direction[1]);
        }
        
        // 返回1表示找到了一个新的岛屿
        // Return 1 indicating a new island has been found
        return 1;
    }
}
```
```

## 复杂度分析

- **时间复杂度**: O(M×N)
    - M和N分别是网格的行数和列数
    - 每个格子最多被访问一次

- **空间复杂度**: O(M×N)
    - visited数组占用O(M×N)空间
    - 最坏情况下递归深度为O(M×N)

## 优化方案

1. **空间优化**：
    - 可以直接修改原数组而不使用visited数组
    - 将访问过的'1'修改为'0'或其他特殊字符

2. **代码优化**：
    - 可以使用BFS替代DFS，在某些情况下可能更快
    - 使用方向数组简化代码
    - 提前处理特殊情况（空数组等）

3. **内存优化**：
    - 使用迭代而不是递归，避免栈溢出
    - 使用队列实现BFS

## 相关题目

- 岛屿的最大面积
- 封闭岛屿的数目
- 飞地的数量
- 统计封闭岛屿的数目

## 总结

本题是经典的DFS应用题，核心在于：
1. 理解岛屿的连通性定义
2. 正确处理边界条件
3. 高效地标记已访问位置
4. 合理运用DFS遍历策略

熟练掌握这类问题的解法对于解决其他图论相关题目很有帮助。