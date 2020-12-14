### 题目链接

[剑指Offer - 04 - 二维数组中的查找](https://leetcode-cn.com/problems/er-wei-shu-zu-zhong-de-cha-zhao-lcof/)

### 题目描述

![image-20201214193542022](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20201214193542.png)

### 解题思路

此题比较重要的就是找到标志位，可以是右上角或者左下角！

**标志数引入**： 此类矩阵中左下角和右上角元素有特殊性，称为标志数。

- 左下角元素： 为所在列最大元素，所在行最小元素。
- 右上角元素： 为所在行最大元素，所在列最小元素。

**标志数性质：** 将 `matrix` 中的**左下角元素**（标志数）记作 `flag` ，则有：

1. 若 `flag > target` ，则 `target` 一定在 `flag` 所在行的上方，即 `flag` 所在行可被消去。
2. 若 `flag < target` ，则 `target` 一定在 `flag` 所在列的右方，即 `flag` 所在列可被消去。

- 本题解以左下角元素为例，同理，**右上角元素** 也具有行（列）消去的性质。

**算法流程：** 根据以上性质，设计算法在每轮对比时消去一行（列）元素，以降低时间复杂度。

1. 从矩阵 matrix 左下角元素（索引设为 `(i, j)` ）开始遍历，并与目标值对比：
   - 当 `matrix[i][j] > target` 时： 行索引向上移动一格（即 `i--`），即消去矩阵第 i 行元素；
   - 当 `matrix[i][j] < target` 时： 列索引向右移动一格（即 `j++`），即消去矩阵第 j 列元素；
   - 当 `matrix[i][j] == target` 时： 返回 `true` 。
2. 若行索引或列索引越界，则代表矩阵中无目标值，返回 false。



```go
func findNumberIn2DArray(matrix [][]int, target int) bool {

    // row控制的是行是要递减的，col控制的是列是要递增的
    // row现在是最大行，col现在是第零列
    // 所以当row < 0 且 col 等于最大列时退出
    // （row是可以为0的，col是不可以为len(matrix[0],j最大是len(matrix[0]) - 1）
    for row, col := len(matrix) - 1, 0; row >= 0 && col < len(matrix[0]);{
        if matrix[row][col] == target {
            return true
        } else if matrix[row][col] > target{ 
            row-- // 此时位置上的数字比target大,则去掉这行,因为这个数字下面都比这个数字大
        } else {
            col++ // 此时位置上的数字比target小,去掉这列,因为这个数字前面的数字都比这个数字小,所以去掉
        }
    }
    return false
}
```



**时空复杂度**

- 时间复杂度：O(n+m)。访问到的下标的行最多增加 n 次，列最多减少 m 次，因此循环体最多执行 n + m 次。
- 空间复杂度：O(1)。

