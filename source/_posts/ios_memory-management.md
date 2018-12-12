---
title: iOS开发 - 对MRC和ARC的理解
categories: iOS开发
tags:
- iOS
- 基础
- 内存管理
top: 100
copyright: ture
---

# 内存管理基本概念
&emsp;&emsp;在OC的内存管理，其实就是引用计数的管理。内存管理就是在程序需要时程序员分配一段内存空间，而当使用完后将它释放。如果对内存资源使用不当，不存会造成内存资源的浪费，甚至会导致程序crash等。
<!-- more -->
# MRC - 手动内存管理
## 引用计数
&emsp;&emsp;我们都知道，当一个对象没有被任何变量引用时，就会被释放。那是怎么知道对象已经没有被任何变量引用了呢？答案就是Objc的引用计数了，相关说明如下：
- 每个对象都有自己的引用计数器，而且是一个整数。
- 任何一个对象，刚创建的时候，引用计数都为1。
- 当结束使用该对象时，其引用计数则减1.
- 当引用计数器为0时，对象占用的内存就会被系统回收，对象将被释放。

&emsp;&emsp;引用计数器的相关操作如下：
- 当对象被创建，即通过alloc/new/copy等方法时，其引用计数器的初始值为1。
- 当给对象发送retain消息时，其引用计数器加1。
- 当给对象发送release消息时，其引用计数器减1。 
- 最后当对象的引用计数器为0时，Objc会给对象发送dealloc消息来销毁对象。

## 内存管理的规律
&emsp;&emsp;在MRC模式下，必须遵循：谁创建、谁释放、谁引用、谁管理的原则。这里用一个简单的🌰来进行说明：
```
// 当对象被创建时，其引用计数器初始值为1
    Dog *dog = [[Dog alloc] init];
    NSLog(@"retainCount = %lu", (unsigned long)[dog retainCount]);  // 1
    
    // 给对象发送retain消息时，其引用计数器加1
    [dog retain];
    NSLog(@"retainCount = %lu", (unsigned long)[dog retainCount]);  // 2
    
    // 给dog指向的Dog实例对象发送一条release消息，其引用计数器会减1
    [dog release];
    NSLog(@"retainCount = %lu", (unsigned long)[dog retainCount]);  // 1
    
    // 若再给dog指向的对象发送release消息，其引用计数器会为0，系统就会释放该对象
    // 最后会调用Dog类的dealloc方法，释放该对象
    [dog release];
```

## 空指针和野指针
### 关于` unrecognized selector sent to instance `的crash
&emsp;&emsp;这是因为对象已经被释放（引用计数为0），这个时候再发消息给这个对象时，就会crash，因为这个时候这个对象就是一个野指针。正确且安全的做法是将对象置为nil，使它成为一个空指针。

### 空指针
- 没有存储任何内存地址的地址就称为空指针（NULL指针）
- 空指针就是被赋值为0的指针，在摸鱼被具体初始化之前，其指为0。

### 野指针
&emsp;&emsp;野指针不是NULL指针，是指向“垃圾”内存（不可用内存）的指针。
&emsp;&emsp;来举个🌰：
```
Cat *cat = [[Cat alloc] init];
[cat setName:@"小猫"];
[cat release];
[cat setName:@"小猫"];
```
&emsp;&emsp;在执行下面代码的时候，最后一行会出现野指针的错误，并且crash。相关说明：
- 假设Cat对象的地址为0xff43，指针变量的地址为0xee44，cat中存储的是Cat对象的地址0xff43，即指针变量cat指向了这个Cat对象。
- 在执行完` [cat release]; `时，给cat指向的Cat对象发送了一条release消息，Cat对象接收到release消息后，会马上被销毁，所占用的内存会被回收。Cat对象被销毁了，地址为0xff43的内存地址就变成了“垃圾内存”，然而指针变量cat仍然指向这一块内存，这时候，cat就称为野指针。
- 在最后一行代码中，cat所指向的Cat对象发送了一条setName:消息，但是Cat对象已经被销毁了，它所占的内存已经是垃圾内存，如果我们还去访问这一块内存，那么就会报野指针错误。
- 因此，要避免野指针错误导致的crash，我们需要在Cat对象被回收之后，将指针变量cat置为nil，变成空指针，没有指向任何对象，因此setName:消息不会发送出去，也不会造成crash。

## 自动释放池
### 简介及原理
&emsp;&emsp;当我们不再使用一个对象的时候，应该将其释放。但我们很难理清一个对象什么时候不再使用，也就不知何时释放，为了解决这个问题，Objc提供了autorelease方法来解决这个问题。
&emsp;&emsp;autorelease实质上是把对release的调用延迟了，对于每一个autorelease，系统只是把该对象放入了当前的autorelease pool中，当改池子被释放时，该池子中的所有对象会被调用release。
&emsp;&emsp;这里需要特别注意：**autorelease方法会返回本身对象，且调用完autorelease方法后，对象的retainCount不变。**

### autorelease的好处
- 不需要关心对象释放的时间
- 不需要关心什么时候、在哪里调用release方法

### autorelease的创建和使用
#### 两种创建方式：
- 使用NSAutoreleasePool进行创建
```
NSAutoreleasePool *pool = [[NSAutoreleasePool alloc] init];
Cat *cat = [[Cat alloc] init];
[cat autorelease];
[pool release];
```

- 使用@autoreleasepool创建
```
@autoreleasepool {
        Cat *cat = [[Cat new] autorelease];
    }
```

#### 使用注意
- 自动释放池实质上只是在释放的时候给池子中所有对象发送release消息，不保证对象一定会销毁，如果自动释放池向对象发送release消息后对象的引用计数仍大于1，那对象就会无法销毁。
- 一些大内存消耗对象的重复创建时，使用@autoreleasepool时，需要将@autoreleasepool放在for循环里面，如果整个for循环是在@autorelease中，则还是会使得内存暴涨。
```
for (int i = 0; i < 99999; ++i) {
    @autoreleasepool {
        NSString *log = [NSString stringWithFormat:@"%d", i];
         NSLog(@"%@", log);
    }
}
```

## MRC下使用ARC
&emsp;&emsp;在项目的Build Phases的Compile Sources中选择需要使用ARC方式的.m文件，然后双击该文件，在弹出的对话框中输入`-fobjc-arc`即可。

# ARC自动内存管理
## 简介
&emsp;&emsp;自动的引用计数（Automatic Referencen Count 简称ARC），苹果在WWDC 2011年大会上提出的用于内存管理的技术。使用ARC后，编译器会我们管理好对象的内存，何时需要保持对象，何时需要自动释放对象，编译器会在合适的地方为我们插入retain、release和autorelease。

## 规则
&emsp;&emsp;只要还有一个强指针变量指向对象，对象就会保持在内存中。

## 强指针和弱指针
- 默认所有实例变量和局部变量都是强指针。
- 弱指针指向的对象被回收后，弱指针会自动变为nil指针，不会引发野指针错误。

## ARC的注意点
- 不允许调用release、retain、autorelease、retainCount方法
- 重写父类的dealloc方法时，不能再调用[super dealloc];
- 两个类相互引用时，其中一个类使用strong，另一个类使用weak

## ARC下使用MRC
&emsp;&emsp;在项目的Build Phases的Compile Sources中选择需要使用MRC方式的.m文件，然后双击该文件，在弹出的对话框中输入`-fno-objc-arc`即可