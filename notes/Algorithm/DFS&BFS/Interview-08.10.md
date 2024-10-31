### [面试题 08.10. 颜色填充](https://leetcode.cn/problems/color-fill-lcci/)
*Easy · DFS · 图像处理*

**题目描述**
实现图像编辑中的颜色填充功能。给定一个二维数组表示的图像，从指定起点(sr,sc)开始，将与起点颜色相同的相连区域都填充为新的颜色。
*Implement a paint fill function of image editing software. Given a 2D array representing an image, starting from point (sr,sc), fill all connected area of the same color with new color.*

**解题思路**
1. 这是一个典型的洪水填充(Flood Fill)问题，可以使用DFS实现
2. 关键是从起点开始，向四个方向扩展，填充所有相连的相同颜色区域
3. 需要记录已访问的位置，避免重复处理
4. 使用方向数组简化四个方向的遍历
5. 递归终止条件：越界、颜色不同或已访问过

**代码实现**
```java
class Solution {
    int rowLen;
    int colLen;
    int originalColor;
    // 定义四个方向：下、上、右、左
    // Define four directions: down, up, right, left
    int[][] directions = {{1, 0}, {-1, 0}, {0, 1}, {0, -1}};
    boolean[][] visited;

    public int[][] floodFill(int[][] image, int sr, int sc, int newColor) {
        // 初始化成员变量
        // Initialize member variables
        rowLen = image.length;
        colLen = image[0].length;
        visited = new boolean[rowLen][colLen];
        originalColor = image[sr][sc];
        
        dfs(image, sr, sc, newColor);
        return image;
    }

    public void dfs(int[][] image, int row, int col, int newColor) {
        // 处理边界条件：越界、颜色不匹配或已访问
        // Handle boundary conditions: out of bounds, color mismatch, or visited
        if (row >= rowLen || row < 0 || col >= colLen || col < 0 
            || image[row][col] != originalColor 
            || visited[row][col]) {
            return;
        }
        
        // 填充新颜色并标记已访问
        // Fill new color and mark as visited
        image[row][col] = newColor;
        visited[row][col] = true;
        
        // 向四个方向递归扩展
        // Recursively expand in four directions
        for (int[] direction : directions) {
            dfs(image, row + direction[0], col + direction[1], newColor);
        }
    }
}
```

**复杂度分析**
- 时间复杂度：O(N×M)，其中N和M是图像的行数和列数，最坏情况下需要访问所有位置
- 空间复杂度：O(N×M)，visited数组的空间开销，递归调用栈的深度最大为N×M

**优化建议**
1. 可以去掉visited数组，直接用newColor标记已访问的位置：
    - 如果新旧颜色相同，可能导致无限递归
    - 如果新旧颜色不同，填充的新颜色本身就起到了标记已访问的作用
2. 可以在主函数中增加新旧颜色相同的判断，相同时直接返回原图像
3. 可以使用队列实现广度优先搜索(BFS)，在某些情况下可能更高效