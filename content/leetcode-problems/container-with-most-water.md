+++
title = '11. 盛最多水的容器'
date = 2024-04-28T22:20:30+08:00
draft = false

+++

## [题目](https://leetcode.cn/problems/container-with-most-water/description/)

给定一个长度为 `n` 的整数数组 `height` 。有 `n` 条垂线，第 `i` 条线的两个端点是 `(i, 0)` 和 `(i, height[i])` 。

找出其中的两条线，使得它们与 `x` 轴共同构成的容器可以容纳最多的水。

返回容器可以储存的最大水量。

**示例 1：**

![img](https://aliyun-lc-upload.oss-cn-hangzhou.aliyuncs.com/aliyun-lc-upload/uploads/2018/07/25/question_11.jpg)

```
输入：[1,8,6,2,5,4,8,3,7]
输出：49 
解释：图中垂直线代表输入数组 [1,8,6,2,5,4,8,3,7]。在此情况下，容器能够容纳水（表示为蓝色部分）的最大值为 49。
```

**示例 2：**

```
输入：height = [1,1]
输出：1
```

**提示：**

- `n == height.length`
- `2 <= n <= 105`
- `0 <= height[i] <= 104`

## 解题思路

面积S = (j - i) * min(h[i], h[j]), j > i
在(1, n)中找出使面积S最大的(i, j)，最简单的方法就是遍历(i, j)的所有可能并比较所有面积。

发现以下规律

1. 当h[i] < h[j]时，不需要计算(i, j-1), (i, j-2), ...，因为它们的面积一定小于(i, j)的面积，因为宽度变小，高度一定<=h[i]。所以下一步应该检查(i+1, j)的面积。
2. 同理h[i] >= h[j]时，不需要计算(i+1, j), (i+2, j), ...， 所以下一步应该检查(i, j-1)的面积。

首尾对撞指针，每次高度较低的指针往中间移动，计算面积更新最大面积。

## 代码

```go
func maxArea(height []int) int {
    l, r := 0, len(height)-1
    ans := 0
    for l < r {
        ans = max(ans, (r-l)*min(height[l], height[r]))
        if height[l] < height[r] {
            l++
        } else {
            r--
        }
    }
    return ans
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

