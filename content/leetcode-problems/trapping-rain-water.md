+++
title = '42. 接雨水'
date = 2024-05-01T11:29:30+08:00
draft = true

+++

## [题目](https://leetcode.cn/problems/trapping-rain-water/)

给定 `n` 个非负整数表示每个宽度为 `1` 的柱子的高度图，计算按此排列的柱子，下雨之后能接多少雨水。

 

**示例 1：**

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/10/22/rainwatertrap.png)

```
输入：height = [0,1,0,2,1,0,1,3,2,1,2,1]
输出：6
解释：上面是由数组 [0,1,0,2,1,0,1,3,2,1,2,1] 表示的高度图，在这种情况下，可以接 6 个单位的雨水（蓝色部分表示雨水）。 
```

**示例 2：**

```
输入：height = [4,2,0,3,2,5]
输出：9
```

 

**提示：**

- `n == height.length`
- `1 <= n <= 2 * 104`
- `0 <= height[i] <= 105`

## 解题思路

第`i`个柱子仅当其左边和右边都存在更高的柱子才能接住水。

计算第`i`个柱子能接住多少水，需要知道其左边最高的柱子和右边最高的柱子，水位取决于两者高度较小值`waterLevel = min(leftMax[i], rightMax[i])`。

如果水位大于当前柱子高度，当前柱子能接住`waterLevel - height[i]`的水，宽度固定为1。

为了快速得到当前柱子其左边最高的柱子和右边最高的柱子，提前计算每个柱子其左边最高的柱子高度`leftMax`数组和其右边最高的柱子高度`rightMax`数组。

时间复杂度：$O(n)$，空间复杂度：$O(n)$

空间优化

由于`leftMax`数组计是从左往右计算，`leftMax`数组是从右往左计算，因此可以使用双指针和两个变量代替两个数组。

维护两个指针`l`和`r`，以及两个变量`leftMax`和`rightMax`，初始时`l = 0, r = n−1,leftMax = 0, rightMax = 0`。

指针`l`只会向右移动，指针`r`只会向左移动，在移动指针的过程中更新两个变量`leftMax`和 `rightMax`的值，即`leftMax = max(leftMax, height[l]), rightMax = max(rightMax, height[r])`。

如果`height[l] < height[r]`，则必定`leftMax < rightMax`，第` l`个柱子的水位确定就是`leftMax`，木桶效应，即便`l`右边有更高的柱子，第`l`个柱子的接水量加到总接水量，然后`l++`。

如果`height[l] >= height[r]`，则必定`leftMax >= rightMax` ，第`r`个柱子的水位确定就是`rightMax`，木桶效应，即便`r`左边有更高的柱子，第`r`个柱子的接水量加到总接水量，然后`r--`。

时间复杂度：$O(n)$，空间复杂度：$O(1)$

## 代码

```go
func trap1(height []int) int {
    n := len(height)
    leftMax := make([]int, n)
    rightMax := make([]int, n)
    for i := 1; i < n; i++ {
        leftMax[i] = max(leftMax[i-1], height[i-1])
    }
    for i := n-2; i >= 0; i-- {
        rightMax[i] = max(rightMax[i+1], height[i+1])
    }
    ans := 0
    // 首尾柱子不能接住水，只有中间柱子能接住水
    for i := 1; i < n-1; i++ {
        waterLevel := min(leftMax[i], rightMax[i])
        if waterLevel > height[i] {
            ans += waterLevel - height[i]
        }
    }
    return ans
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func trap(height []int) int {
    n := len(height)
    leftMax, rightMax := 0, 0
    l, r := 0, n-1
    ans := 0
    for l <= r {
        leftMax = max(leftMax, height[l])
        rightMax = max(rightMax, height[r])
        if height[l] < height[r] {
            ans += leftMax - height[l]
            l++
        } else {
            ans += rightMax - height[r]
            r--
        }
    }
    return ans
}
```



