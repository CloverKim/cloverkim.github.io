---
title: 算法学习 - 冒泡排序
categories: 算法学习
tags:
- 算法
top: 100
copyright: ture
---

# 原理
1. 比较相邻的元素，如果第一个比第二个大，就交换他们两个。
2. 对每一组相邻元素做同样的工作，从开始第一对到结尾的最后一对。在这一点，最后的元素应该会是最大的数。
3. 针对所有的元素重复以后的步骤，除了最后一个。
4. 持续每次对越来越少的元素重复上面的步骤，直到满意任何一对数字需要比较，则序列最终有序。
![](https://ws1.sinaimg.cn/large/749c46aagy1fvrm3x9pj5j20iu08xq4b.jpg '步骤图')

# Swift实现
```
func bubble_sort(_ array: inout [Int]) {
    for i in 0..<array.count - 1 {
        for j in 0..<array.count - i - 1 {
            if array[j] > array[j + 1] {
                array.swapAt(j, j + 1)
            }
        }
    }
}

var arrays: [Int] = [3, 6, 4, 2, 9, 8, 5]
bubble_sort(&arrays)
```
# 时间复杂度
冒泡排序的时间复杂度为O(n^2)