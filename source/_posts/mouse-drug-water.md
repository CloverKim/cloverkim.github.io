---
title: 一道有趣的算法题：15个瓶子，4只老鼠，测试哪瓶有毒？
categories: 算法学习
tags:
- 算法
- 位运算
top: 100
copyright: ture
---

# 请听题
> 有15个瓶子，其中最多有一瓶有毒，现在有四只老鼠，喝了有毒的水之后，第二天就会死。如何在第二天就可以判断出哪个瓶子有毒？
<!-- more -->

# 基本思路
&emsp;&emsp;用二进制给每个瓶子进行编码0001-1111，如下图所示。四只老鼠分别喝第n位是1的瓶子的水，比如第1只老鼠喝1（0001）、3（0011）、5（0101）、7（0111）、9（1001）、11（1011）、13（1101）、15（1111）号瓶子的水。第二天，再通过4只老鼠的存活状态的组合来判断具体哪一瓶有毒。对于老鼠的存活状态，有生和死两种状态，0表示存活，1表示死亡，比如0011，即第1、2只老鼠死亡，第3、4只老鼠存活，0011进制表示的瓶子为3号瓶子，因此3号瓶子有毒。具体组合和结果如下图所示。
![](http://pz1livcqe.bkt.clouddn.com/瓶子编号.jpg '瓶子编号')
![](http://pz1livcqe.bkt.clouddn.com/老鼠存活组合.jpg '老鼠存活组合')

# 代码实现
```
- (void)drugWater:(NSString *)mice {
    NSInteger drug = 0;     // 有毒的瓶子编号
    int mices[miceNum] = {0};
    
    for (NSInteger i = 0; i < [mice length]; i++) {
        mices[i] = [[mice substringWithRange:NSMakeRange(i, 1)] intValue];
    }
    
    for (NSInteger i = 0; i < miceNum; i++) {
        drug |= (mices[i] << (miceNum - i - 1));
    }
    
    if (drug == 0) {
        NSLog(@"%@", @"无毒");
    } else {
        NSLog(@"有毒的瓶子是第 %ld 瓶", (long)drug);
    }
}
```
相关说明：
- 方法传的mice参数为，4只老鼠的存活状态的字符串，比如0011。
- 根据之前的分析结果，我们用位运算来得出有毒的瓶子号即可。