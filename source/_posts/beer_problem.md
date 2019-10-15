---
title: 一道有趣的算法题：10元最多可喝多少瓶啤酒？
categories: 算法学习
tags:
- 算法
- 递归
top: 100
copyright: ture
---

# 请听题
> 每瓶啤酒2元，2个空酒瓶或4个瓶盖可换1瓶啤酒。10元最多可喝多少瓶啤酒？

# 网上的解答
关于答案：网上有非常多的解答。包括：
- 一步步进行推算
- 设一瓶酒里的酒价值x，酒瓶价值y，瓶盖价值z，列等式进行计算
- 跟老板赊账或者借酒瓶或者瓶盖计算
- ...等等
<!-- more -->
以下是网上给的实际演算过程：
1、用10元，先买回来5瓶酒，喝完后剩5个瓶子和5个盖子；（喝5瓶）
2、拿4个瓶子，4个盖子换回3瓶酒，喝完后剩下4个瓶子和4个盖子；（喝3瓶）
3、拿4个瓶子，4个盖子换回3瓶酒，喝完后剩下3个瓶子和三个盖子；（喝3瓶）
4、拿2个瓶子换1瓶酒，喝完后剩下2个瓶子，4个盖子；（喝1瓶）
5、拿2个瓶子，4个盖子换2瓶酒，喝完后剩下2个瓶子，2个盖子；（喝2瓶）
6、拿2个瓶子换1瓶酒，喝完后剩1个瓶子，3个盖子。（喝1瓶）
一共能够喝15瓶啤酒，但是还剩余1个瓶子和3个瓶盖。

# 采用递归算法实现 - Swift
作为一名程序猿，用笔一步步计算太麻烦了，知道原理，那就写个递归算法计算一波。
```
var drinkTotal = 5          // 总共喝的啤酒的数量,初始化为5，即购买的5瓶
var leftBottle = 0          // 剩余空瓶的数量
var leftCover = 0           // 剩余瓶盖的数量

func convert(_ bottle: Int, _ cover: Int) -> Int {
    if bottle >= 2 || cover >= 4 {
        leftBottle = (bottle / 2) + (bottle % 2) + (cover / 4)
        leftCover = (cover / 4) + (cover % 4) + (bottle / 2)
        return (bottle / 2) + (cover / 4) + convert(leftBottle, leftCover)
    }
    return 0
}

drinkTotal += convert(5, 5)
print("\n总共喝了\(drinkTotal)瓶啤酒，还剩下\(leftBottle)个空瓶，\(leftCover)个空盖")
```
下面的写法更加通俗易懂
```
/// 啤酒信息类
class TurnBeerInfo {
    var bottleTotal = 0       // 空瓶的数量
    var coverTotal = 0        // 瓶盖的数量
    var drinkTotal = 0        // 已喝啤酒的数量
}

/// 购买啤酒
func buyBeer(_ money: Int, _ turnBeerInfo: TurnBeerInfo) {
    let beerTotal = money / 2
    turnBeerInfo.bottleTotal = beerTotal
    turnBeerInfo.coverTotal = beerTotal
    turnBeerInfo.drinkTotal = beerTotal
}

/// 兑换啤酒
func convert(_ turnBeerInfo: TurnBeerInfo, _ i: Int) {
    // 用空瓶兑换啤酒的数量
    let bottleTurnNum = turnBeerInfo.bottleTotal / 2
    // 计算空瓶兑换后的瓶盖数量
    turnBeerInfo.coverTotal += bottleTurnNum
    
    // 用瓶盖兑换啤酒的数量
    let coverTurnNum = turnBeerInfo.coverTotal / 4
    // 重新计算瓶盖兑换后的瓶盖数量
    turnBeerInfo.coverTotal = turnBeerInfo.coverTotal % 4 + coverTurnNum
    // 重新计算瓶盖兑换后的空瓶数量
    turnBeerInfo.bottleTotal = turnBeerInfo.bottleTotal % 2 + bottleTurnNum + coverTurnNum
    // 计算总共喝了几瓶
    turnBeerInfo.drinkTotal = turnBeerInfo.drinkTotal + coverTurnNum + bottleTurnNum
    
    print("这是第\(i)次兑换，目前已喝了\(turnBeerInfo.drinkTotal)瓶啤酒，还剩下\(turnBeerInfo.bottleTotal)个空瓶，\(turnBeerInfo.coverTotal)个瓶盖")

    // 满足条件，继续计算
    if turnBeerInfo.bottleTotal >= 2 || turnBeerInfo.coverTotal >= 4 {
        let j = i + 1
        convert(turnBeerInfo, j)
    }
}

let turnBeerInfo = TurnBeerInfo()
let totalMoney = 10
buyBeer(totalMoney, turnBeerInfo)
convert(turnBeerInfo, 1)
print("\n总共喝了\(turnBeerInfo.drinkTotal)瓶啤酒，还剩下\(turnBeerInfo.bottleTotal)个空瓶，\(turnBeerInfo.coverTotal)个空盖")
```

# 运行结果
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fwbncmbf1qj20aj03874o.jpg)