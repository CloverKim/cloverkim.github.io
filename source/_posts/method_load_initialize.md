---
title: iOS开发 - +load 和 +initialize 方法
categories: iOS开发
tags:
- iOS
- Objective-C
- Swift 
top: 100
copyright: ture
---

# 概述
&emsp;&emsp;Objective-C作为一门面向对象的开发语言，有类和对象的概念。编译后，类相关的数据结构会保留在目标文件中，在运行时得到解析和使用。在应用程序运行起来的时候，类的信息会有加载和初始化的过程。
&emsp;&emsp;就像Application有生命周期回调方法一样，在Objective-C的类被加载和初始化的时候，也可以收到方法回到，可以在适当的情况下做一些定制处理。而这正是load和initialize方法可以帮我们做到的。
```
+ (void)load;
+ (void)initialize;
```
<!-- more -->
&emsp;&emsp;可以看到这两个方法都是以“+”开头的类方法，返回值为空。通常情况下，我们在开发过程中可能不必关注这两个方法。如果有需要定制，我们可以在自定义的NSObject子类中给出这两个方法的实现，这样在类的加载和初始化过程中，自定义的方法可以得到调用。

# +load方法
&emsp;&emsp;顾名思义，+load方法是在这个文件被程序装载时调用。只要是在Compile Sources中出现的文件总是会被装载，这与这个类是否被用到无关，因此+load方法总在main函数之前调用。
&emsp;&emsp;下面是Apple文件的相关描述：
> Invoked whenever a class or category is added to the Objective-C runtime; implement this method to perform class-specific behavior upon loading.
> Discussion
> The load message is sent to classes and categories that are both dynamically loaded and statically linked, but only if the newly loaded class or category implements a method that can respond.
> The order of initialization is as follows:
> 1. All initializers in any framework you link to.
> 2. All +load methods in your image.
> 3. All C++ static initializers and C/C++ __attribute__(constructor) functions in your image.
> 4. All initializers in frameworks that link to you.
> In addition:
> 1. A class’s +load method is called after all of its superclasses’ +load methods.
> 2. A category +load method is called after the class’s own +load method.
> In a custom implementation of load you can therefore safely message other unrelated classes from the same image, but any load methods implemented by those classes may not have run yet.

## load函数调用特点如下：
&emsp;&emsp;当类被引用进项目的时候就会执行+load方法（在main函数开始执行之前），与这个类是否被用到无关，每个类的+load方法**只会自动调用一次**。由于+load方法是系统自动加载的，因此不需要调用父类的+load方法，否则父类的+load方法会执行多次。
- 当父类和子类都实现+load方法时，父类的+load方法执行顺序要优先于子类。
- 当子类未实现+load方法时，不会调用父类的+load方法。
- 类中的+load方法执行顺序要优先于类别(Category)
- 当有多个类别(Category)都实现了+load方法，这几个+load方法都会执行，但执行的顺序不确定，其执行的顺序与类别在Compile Sources中出现的顺序一致。
- 当然当有多个不同的类的时候，每个类的+load方法执行的顺序与其在Compile Sources出现的顺序一致。

## 下面通过🌰来验证
&emsp;&emsp;新建3个类：Animal继承NSObject，Cat和Dog均继承Animal，并且新建3个Animal的分类，分别命名为Animal (Category1)、Animal (Category2)、Animal (Category3)
在Animal、Cat、Animal (Category1)、Animal (Category2)、Animal (Category3)均实现+load方法，并且打印对应的信息（Dog类除外）
```
+ (void)load {
    NSLog(@"%s", __func__);
}
```
运行结果如下图所示：
![](https://ws1.sinaimg.cn/large/749c46aagy1fwdp9n7oruj20gm04675h.jpg '+load')
由上图的运行结果可知：
- 首先执行的父类Animal的load方法，再执行子类Cat的load方法，说明父类的load方法执行顺序要优先于子类。
- 子类Dog中没有实现load方法，没有打印对应的信息，说明子类没有实现load方法时，并不会调用父类的load方法。
- 然后执行的是3个分类的load方法，并且没有顺序，说明分类中的load方法的执行要晚于类的load方法，在多个分类中，只要实现了load方法，都会执行，但执行的顺序不确定。
- 最后执行的是main函数，说明在没有对类进行任何操作的情况下，load方法会被默认执行，并且是在main函数之前。