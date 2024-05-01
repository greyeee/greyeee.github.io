+++
title = '18. 四数之和'
date = 2024-05-01T10:14:47+08:00
draft = false

+++

## [题目](https://leetcode.cn/problems/4sum/)

给你一个由 `n` 个整数组成的数组 `nums` ，和一个目标值 `target` 。请你找出并返回满足下述全部条件且**不重复**的四元组 `[nums[a], nums[b], nums[c], nums[d]]` （若两个四元组元素一一对应，则认为两个四元组重复）：

- `0 <= a, b, c, d < n`
- `a`、`b`、`c` 和 `d` **互不相同**
- `nums[a] + nums[b] + nums[c] + nums[d] == target`

你可以按 **任意顺序** 返回答案 。

**示例 1：**

```
输入：nums = [1,0,-1,0,-2,2], target = 0
输出：[[-2,-1,1,2],[-2,0,0,2],[-1,0,0,1]]
```

**示例 2：**

```
输入：nums = [2,2,2,2,2], target = 8
输出：[[2,2,2,2]] 
```

**提示：**

- `1 <= nums.length <= 200`
- `-109 <= nums[i] <= 109`
- `-109 <= target <= 109`

## 解题思路

方法一：排序+双指针，在[3Sum](/leetcode-problems/3sum)的基础上多套一层循环。第一层循环固定第一个元素，第二层循环固定第二个元素，循环内部用对撞指针找出剩余两个元素。时间复杂度：$O(n^3)$，空间复杂度：$O(logn)$，取决于排序所需的空间。

方法二：分治法，将问题分成若干子问题，对子问题求解后将解合并。4Sum -> n-3个3Sum，3Sum -> n-2个2Sum，2Sum用对撞指针得到解。延展到求kSum。

## 代码

方法一

```go
func fourSum(nums []int, target int) [][]int {
    var ans [][]int
    n := len(nums)
    sort.Ints(nums)
    for i := 0; i < n-3; i++ {
        if i > 0 && nums[i] == nums[i-1] {
            continue
        }
        for j := i+1; j < n-2; j++ {
            if j > i+1 && nums[j] == nums[j-1] {
                continue
            }
            l, r := j+1, n-1
            for l < r {
                s := nums[i] + nums[j] + nums[l] + nums[r]
                if s == target {
                    ans = append(ans, []int{nums[i], nums[j], nums[l], nums[r]})
                    lv, rv := nums[l], nums[r]
                    for l < r && nums[l] == lv {
                        l++
                    }
                    for l < r && nums[r] == rv {
                        r--
                    }
                } else if s < target {
                    l++
                } else {
                    r--
                }
            }
        }
    }
    return ans
}
```

方法二

```go
func fourSum(nums []int, target int) [][]int {
    sort.Ints(nums)
    return kSum(nums, target, 4)
}

func kSum(nums []int, target int, k int) [][]int {
    // 剪枝
    if !(nums[0]*k <= target && target <= nums[len(nums)-1]*k) {
        return [][]int{}
    }
    
    if k == 2 {
        return twoSum(nums, target)
    }
    
    var ans [][]int
    for i := 0; i < len(nums)-k+1; i++ {
        if i > 0 && nums[i] == nums[i-1] {
            continue
        }
        for _, res := range kSum(nums[i+1:], target-nums[i], k-1) {
            ans = append(ans, append([]int{nums[i]}, res...))
        }
    }
    return ans
}

func twoSum(nums []int, target int) [][]int {
    var ans [][]int
    l, r := 0, len(nums)-1
    for l < r {
        s := nums[l] + nums[r]
        if s == target {
            ans = append(ans, []int{nums[l], nums[r]})
            lv, rv := nums[l], nums[r]
            for l < r && nums[l] == lv {
                l++
            }
            for l < r && nums[r] == rv {
                r--
            }
        } else if s < target {
            l++
        } else {
            r--
        }
    }
    return ans
}
```

