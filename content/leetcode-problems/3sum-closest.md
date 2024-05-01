+++
title = '16. 最接近的三数之和'
date = 2024-05-01T09:54:21+08:00
draft = false

+++

## [题目](https://leetcode.cn/problems/3sum-closest/)

给你一个长度为 `n` 的整数数组 `nums` 和 一个目标值 `target`。请你从 `nums` 中选出三个整数，使它们的和与 `target` 最接近。

返回这三个数的和。

假定每组输入只存在恰好一个解。

**示例 1：**

```
输入：nums = [-1,2,1,-4], target = 1
输出：2
解释：与 target 最接近的和是 2 (-1 + 2 + 1 = 2) 。
```

**示例 2：**

```
输入：nums = [0,0,0], target = 1
输出：0
```

**提示：**

- `3 <= nums.length <= 1000`
- `-1000 <= nums[i] <= 1000`
- `-104 <= target <= 104`

## 解题思路

排序+双指针，和[3Sum](/leetcode-problems/3sum)思路一样，只是求的是最接近target的三数之和，维护最接近的和`ans`和最小差值`minDiff`，当总和更靠近时更新`ans`和`minDiff`。

## 代码

```go
func threeSumClosest(nums []int, target int) int {
    sort.Ints(nums)
    ans, minDiff := 0, math.MaxInt32
    for i := 0; i < len(nums)-2; i++ {
        if i > 0 && nums[i] == nums[i-1] {
            continue
        }
        
        l, r := i+1, len(nums)-1
        for l < r {
            s := nums[i] + nums[l] + nums[r]
            if abs(s - target) < minDiff {
                ans, minDiff = s, abs(s - target)
            }
            if s == target {
                return ans
            } else if s < target {
                l++
            } else {
                r--
            }
        }
    }
    return ans
}

func abs(a int) int {
    if a < 0 {
        return -a
    }
    return a
}
```

