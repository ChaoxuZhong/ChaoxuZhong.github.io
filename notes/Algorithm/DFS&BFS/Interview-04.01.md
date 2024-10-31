### [面试题 04.01. 节点间通路](https://leetcode.cn/problems/route-between-nodes-lcci/)
*Medium · 图论 · DFS*

**题目描述**
在有向图中判断是否存在一条从起点到终点的路径。图中可能包含重边和自环。
*Given a directed graph, design an algorithm to find out whether there is a route between two nodes. The graph may contain parallel edges and self-loops.*

**解题思路**
1. 使用邻接表表示有向图，便于查找节点的所有出边
2. 采用DFS搜索，从起点开始探索所有可能路径
3. 使用visited数组避免重复访问，防止环路导致的死循环
4. 当访问到目标节点时立即返回true
5. 关键点是处理图中的特殊情况（重边和自环）

**代码实现**
```java
class Solution {
    // 邻接表存储图结构
    // Adjacency list to store graph structure
    Map<Integer, List<Integer>> graphMap = new HashMap<>();
    boolean[] visited;
    
    public boolean findWhetherExistsPath(int n, int[][] graph, int start, int target) {
        // 构建邻接表，处理重边自然合并到List中
        // Build adjacency list, parallel edges are naturally merged into the List
        for (int[] gItem : graph) {
            graphMap.computeIfAbsent(gItem[0], k -> new ArrayList<Integer>())
                   .add(gItem[1]);
        }
        
        // 初始化访问数组
        // Initialize visited array
        visited = new boolean[n];
        return dfs(start, target);
    }
    
    // DFS查找目标节点
    // DFS to find target node
    public boolean dfs(int cur, int target) {
        // 找到目标节点，返回true
        // Found target node, return true
        if (cur == target) return true;
        
        // 当前节点已访问或无出边，返回false
        // Current node is visited or has no outgoing edges, return false
        if (visited[cur] || graphMap.get(cur) == null) return false;
        
        // 标记当前节点已访问
        // Mark current node as visited
        visited[cur] = true;
        
        // 遍历所有相邻节点
        // Traverse all adjacent nodes
        for (Integer next : graphMap.get(cur)) {
            if (dfs(next, target)) {
                return true;
            }
        }
        return false;
    }
}
```

**复杂度分析**
- 时间复杂度：O(V + E)，其中V是节点数，E是边数
    - 构建邻接表需要O(E)
    - DFS最多访问所有节点和边一次
- 空间复杂度：O(V + E)
    - 邻接表存储需要O(V + E)
    - visited数组需要O(V)
    - 递归调用栈最深为O(V)

**优化建议**
1. 可以使用BFS替代DFS，在某些情况下可能更快找到路径
2. 如果需要频繁查询，可以使用Floyd算法预处理出任意两点间是否有路径
3. 对于稀疏图，可以考虑使用邻接矩阵代替邻接表
4. 可以在遍历边时进行去重，减少存储空间