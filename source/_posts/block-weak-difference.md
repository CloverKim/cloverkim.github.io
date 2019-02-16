---
title: iOS开发 - __weak和__block的理解
categories: iOS开发
tags:
- iOS
- 基础
top: 100
copyright: ture
---

# __block
&emsp;&emsp;在block里可以访问局部变量，但不能修改局部变量。这是因为当局部变量在block中使用时，实际上使用的变量是在block中复制的数据，所以在block中修改的变量并不能修改block外面的变量值。这里要注意的是可变数组或者字典在block中添加或删除数据时，并不用`__block`修饰，因为在block里使用这些数组时，数组的指针并没有发生变化，仅仅是内存的内容发生了变化。<!-- more -->
关于__block在MRC和ARC模式下的区别：
- `__block`在MRC下有两个作用：
    - 允许在block内访问和修改局部变量
    - 禁止block对所引用的对象进行隐式strong操作
- `__block`在ARC下的作用：
    - 允许在block内访问和修改局部变量

# __weak
&emsp;&emsp;在block中，block会对其对象强引用，对于self也会形成强引用，而self本身对于block也是强引用，这样就会造成循环引用的问题，这时候就需要用`__weak`打破循环，使对象弱引用。或者在block执行完后，将block置为nil也可以打破循环引用，但是该方法会使block只会执行一次，要是再次使用的话，就要重新赋值。

# 区别
- `__block`不管是ARC还是MRC下都能使用，可以修饰对象，还可以修饰基本数据类型。
- `__weak`只能在ARC下使用，只能修饰对象，不能修饰基本数据类型。
- `__block`对象可以在block中被重新赋值，`__weak`不可以。
- `__block`对象在ARC下可能会导致循环引用，非ARC下会避免循环引用；`__weak`只能在ARC下使用，可以避免循环引用。