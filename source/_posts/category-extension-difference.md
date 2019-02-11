---
title: iOS开发 - Category和Extension的区别
categories: iOS开发
tags:
- iOS
- 基础
top: 100
copyright: ture
---

# 区别一
- Category
    - 专门用来给类添加新的方法
    - 不能给类添加成员属性（其实是可以通过runtime给分类添加属性）
    - 分类中用`@property`定义变量，只会生成变量的getter、setter方法的声明，不能生成方法实现和带下划线的成员变量。
<!-- more -->
- Extension
    - 可以说是特殊的分类，也称作匿名分类
    - 可以给类添加成员属性，但是是私有变量
    - 可以给类添加方法，也是私有方法

# 区别二
&emsp;&emsp;虽然有人说Extension是一个特殊的Category，也有人将Extension成为匿名分类，但是两者的区别很大。
- Category
    - 是运行期决定的
    - 类扩展可以添加实例变量，分类不能添加实例变量（原因：因为在运行期，对象的内存布局已经确定，如果添加实例变量会破坏类的内部布局，这对编译性语言是灾难性的。）

- Extension
    - 在编译器决定，是类的一部分，在编译器和头文件的`@interface`和实现文件里的`@implement`一起形成了一个完整的类。
    - 伴随着类的产生而产生，也随着类的消失而消失。
    - Extension一般用来隐藏类的私有消息，必须有一个类的源码才能添加一个类的Extension，所以对于系统的一个类，比如NSString，就无法添加类扩展。
    