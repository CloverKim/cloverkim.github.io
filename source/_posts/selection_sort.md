---
title: 算法学习 - 选择排序
categories: 算法学习
tags:
- 算法
top: 100
copyright: ture
---

# 原理
&emsp;&emsp;初始时在序列中找到最小（大）的元素，放到序列的起始位置作为已排序序列；然后再从剩余未排序元素中继续寻找最小（大）元素，放到已排序序列的末尾。直到所有元素均排列完毕。
<!-- more -->
# Swift 实现
```
func selection_sort(_ array: inout [Int]) {
    var minIndex = 0
    for i in 0..<array.count - 1 {
        minIndex = i
        for j in (i + 1)..<array.count {
            if array[j] < array[minIndex] {
                minIndex = j
            }
        }
        if minIndex != i {
            array.swapAt(minIndex, i)
        }
    }
}

var selectionArrays: [Int] = [3, 6, 4, 2, 9, 8, 5]
selection_sort(&selectionArrays)
```
<br>
# 和冒泡排序的区别
&emsp;&emsp;冒泡排序通过依次交换相邻两个顺序不符合的元素位置，从而将当前最小（大）的元素放到合适的位置；而选择排序每次遍历依次都记住了当前最小（大）元素的位置，最后仅需一次交换操作即可将其放到合适的位置。

# 时间复杂度
&emsp;&emsp;比较次数O(n^2 ) 第一次内循环比较n-1次，然后是n-2次，n-3次，...，最后一次内循环比较1次。共比较的次数是(n-1) + (n-2) + ... + 1，求等差数列和，得n * (n - 1) / 2。其时间复杂度为O(n^2 )。