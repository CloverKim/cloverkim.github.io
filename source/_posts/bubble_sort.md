---
title: 算法学习 - 冒泡排序
categories: 算法学习
tags:
- 算法
top: 100
copyright: ture
---

## 原理
1. 比较相邻的元素，如果第一个比第二个大，就交换他们两个。
2. 对每一组相邻元素做同样的工作，从开始第一对到结尾的最后一对。在这一点，最后的元素应该会是最大的数。
 <!-- more -->
3. 针对所有的元素重复以后的步骤，除了最后一个。
4. 持续每次对越来越少的元素重复上面的步骤，直到满意任何一对数字需要比较，则序列最终有序。
![](http://pic.cloverkim.com/749c46aagy1fvrm3x9pj5j20iu08xq4b.jpg '步骤图')

## Swift实现
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
### 代码优化：
在某些情况下，循环还未终止，整个数组已经排好序，此时应及时终止循环。（冒泡每次都会比较相邻两个数并交换次数不对的组，若一次循环后，都没进行交换，则判断为已完成排序）
### 优化代码实现：
```
func bubble_sort(_ array: inout [Int]) {
    var isChanged = false
    for i in 0..<array.count - 1 {
        isChanged = false
        for j in 0..<array.count - i - 1 {
            if array[j] > array[j + 1] {
                array.swapAt(j, j + 1)
                isChanged = true
            }
        }
        if (!isChanged) {
            break
        }
    }
}
```
## 时间复杂度
- 冒泡排序的时间复杂度为O(n^2)