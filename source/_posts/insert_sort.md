---
title: 算法学习 - 插入排序
categories: 算法学习
tags:
- 算法
top: 100
copyright: ture
---

# 原理
&emsp;&emsp;把待排序的记录按其关键码值的大小逐个插入到一个已经排好序的有序序列中，知道所有的记录插入完为止，得到一个新的有序序列。

算法思路：
1. 设置监视哨r[0]，将待插入记录的值赋值给r[0]
2. 设置开始查找的位置j
3. 在数组中进行搜索，搜索中将j个记录后移，直到r[0]的值大于或等于r[j]的值为止。
4. 将r[0]插入到r[j+1]的位置上。
<!-- more -->
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fw0m1eh8vjj20dw06xaaz.jpg '插入排序') 

# Swift 实现
```
func insert_sort(_ array: inout [Int]) {
    for i in 0..<array.count {
        let key = array[i]
        var j = i - 1
        while j >= 0 {
            if array[j] > key {
                array[j + 1] = array[j]
                array[j] = key
            }
            j -= 1
        }
    }
}

var arrays: [Int] = [3, 6, 4, 2, 9, 8, 5]
insert_sort(&arrays)
```

# 改进
* 在插入某个元素之前需要先确定该元素在有序数组中的位置，当数据量比较大的时候，这是一个很耗时的过程，可以采用二分查找法改进，称为“二分插入排序”。
* 另外一种是在二分插入排序的基础上进一步改进的排序，称为2-路插入排序。其目的是减少排序过程中移动记录的次数，但为此需要n个记录的辅助空间。

# 算法复杂度
&emsp;&emsp;最好的情况是，序列已经是升序排列，在这种情况下，需要进行比较操作需n-1次即可。最坏情况就是，序列是降序排列，那么此时需要进行的比较共有n(n-1)/2。平均来说插入排序算法的时间复杂度为O(n^2 )。