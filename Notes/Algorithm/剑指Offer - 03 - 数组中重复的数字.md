### 题目链接

[剑指Offer - 03 - 数组中重复的数字](https://leetcode-cn.com/problems/shu-zu-zhong-zhong-fu-de-shu-zi-lcof/)

### 题目描述

![题目描述](https://garmen-imgsubmit.oss-cn-shenzhen.aliyuncs.com/img/20200522174447.png)



### 解题思路

1. 这种题其实可以使用`HashMap`进行解决，不过会浪费空间。
2. 也可以先排序之后再查找（先排序排序的目的就是，所有相同的数字会挨到一起，这样，就只需要和自己的下一位比较就能获取到重复值）。
3. 但是能够原地就最好了，原地的思路就是，巧用数组性质。

**1.排序+遍历**

```go
func findRepeatNumber(nums []int) int {
    // 使用内置函数排序
    sort.Ints(nums)
    // 取首元素
    LASTNum := num[0]
    // 从第一位开始遍历（首位是第零位）
    for _, num := range num[1:] {
        if lastNum == num {
            break
        } else {
            lstNum = num //把值赋值给laskNum让它和下一位相比较
        }
    }
    return lastNum
}
```

**2.使用字典（这种比较容易想得到）**

```go
func findRepeatNumber(nums []int) int {
    maps := make(map[int]bool)
    for _, num range nums {
        if maps[num] {
            return num
        } else {
            map[num] = true
        }
    }
    return -1
}
```



**3.巧用数组的性质**

首先根据题目可以得出,长度为n的数组,第二它的元素在`0-n-1`之间取值,这就使得下面的算法得以实行.也就是可以使用数组的性质.
我测试的数据`[2, 3, 1, 0, 4, 3]` 长度为5的数组,它的最大值只能为4,否则算法可能会失效
假设数组中不存在重复数字，则数组排序后第i个位置的数字为`i`，数组中有重复数字，则导致某些数字出现在多个位置。

具体为：从数组首端开始扫描数组，对于下标为`i`的数字`nums[i]`，如果其等于下标i，则扫描下一个数字。如果不等，则把它和下标为`nums[i]`的数字`nums[nums[i]]`比较，如果两者相等，则说明`nums[i]`为重复数字，返回其值即可；如果不等，就把第`i`个数字和第`nums[i]`个数字交换，这样第`nums[i]`个数字对应的就是`nums[i]`，接下来再重复这个交换比较的过程。

```go
func findRepeatNumber(nums []int) int {
    // 使用切片下标index和值num对切片进行遍历
    for index, num := range nums{ 	
        // 如果数组下标和当前遍历的value相等，则继续遍历（不会往下！），已经归位了
        if index == num {
            continue
        }
        // 出现了重复！则证明是我们想要找的
        if nums[num] == num {
            return num
        }
        // 如果元素对应的位置上的值不等于该元素的索引且也不重复，则进行位置交换
        num[num], num[index] = nums[index], num[num]
    }
    return 0
}
```







