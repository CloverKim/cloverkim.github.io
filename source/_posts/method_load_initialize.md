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
&emsp;&emsp;下面是Apple文档的相关描述：
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
![](http://pz1livcqe.bkt.clouddn.com/load.jpg '+load')

由上图的运行结果可知：
- 首先执行的父类Animal的load方法，再执行子类Cat的load方法，说明父类的load方法执行顺序要优先于子类。
- 子类Dog中没有实现load方法，没有打印对应的信息，说明子类没有实现load方法时，并不会调用父类的load方法。
- 然后执行的是3个分类的load方法，并且没有顺序，说明分类中的load方法的执行要晚于类的load方法，在多个分类中，只要实现了load方法，都会执行，但执行的顺序不确定。
- 最后执行的是main函数，说明在没有对类进行任何操作的情况下，load方法会被默认执行，并且是在main函数之前。

## 应用场景
&emsp;&emsp;从上面的分析中，main函数是整个应用程序的入口，而+load方法是在main函数之前执行的，并且只执行一次，所以使用时需要注意。
- 不要做耗时操作
&emsp;&emsp;因为在执行main函数之前，所有实现的load方法执行完才会启动应用，因此在load方法中执行耗时操作时，必然会影响程序的启动时间。
- 不要做对象的初始化操作
&emsp;&emsp;因为在main函数之前自动调用，load方法调用的时候使用者根本就不能确定自己要使用的对象是否已经加载进来了，所以千万不能再这里初始化对象。
- 常用场景是在load方法中实现Method Swizzle
&emsp;&emsp;Method Swizzing是发生在运行时，主要用户在运行时将两个Method进行交换。

# +initialize方法
&emsp;&emsp;下面是Apple文档的相关描述：
> Initializes the class before it receives its first message.
> Discussion
> The runtime sends initialize to each class in a program just before the class, or any class that inherits from it, is sent its first message from within the program. Superclasses receive this message before their subclasses.

> The runtime sends the initialize message to classes in a thread-safe manner. That is, initialize is run by the first thread to send a message to a class, and any other thread that tries to send a message to that class will block until initialize completes.
> ...

## initialize函数调用特点如下：
&emsp;&emsp;initialize在类或者其子类的第一个方法被调用前调用。即使类文件被引用进项目，但是没有使用，initialize不会被调用。由于是系统自动调用，也不需要再调用[super initialize]，否则父类的initialize方法会被多次执行。
- 父类的initialize方法会比子类先执行
- 当子类未实现initialize方法时，会调用父类的initialize方法。
- 当有多个Category都实现了initialize方法，会覆盖类中的方法，只执行一个（会执行Compile Sources列表中最后一个Category的initialize方法）

## 下面通过🌰来验证
&emsp;&emsp;依旧使用上面创建的3个类：Animal、Cat和Dog。
我们在Animal类中实现+initialize方法：
```
+ (void)initialize {
    NSLog(@"%s", __func__);
}
```
运行结果如下图所示：
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fwgtelwx4zj20fu02jaab.jpg)
由运行结果发现，Animal类的+initialie方法被调用了2次，这是因为在创建子类对象时，首先要创建父类对象，所以会调用一次父类的initialize方法，然后创建子类时，尽管该类没有实现initialize方法，但还是会调用父类的方法。虽然initialize方法对一个类而言会调用一次，但这里由于出现了两个类，所以调用两次符合规则，但不符合我们的需求。正确使用initialize方法的方式如下：
```
+ (void)initialize {
    if (self == [Animal class]) {
        NSLog(@"%s", __func__);
    }
}
```
加上判断后，就不会因为子类而调用到自己的initialize方法了。

## 应用场景
&emsp;&emsp;initialize方法主要用来对一些不方便在编译期初始化的对象进行赋值。比如NSMutableArray这种类型的实例化依赖于Runtime的消息发送，所以无法再编译期初始化。

# 总结
- load和initialize方法都会在实例化对象之前调用，以main函数为分水岭。load方法是在main函数之前调用，而initi方法是在main函数之后调用。这两个方法都会被自动调用，无需手动调用。
- load和initialize方法都不用调用父类的方法，即使子类没有initialize方法也会调用父类的方法，而load方法则不会调用父类的load方法。
- load方法通常用来进行Method Swizzle，initialize方法一般用于初始化全局变量或静态变量。
- load和initialize方法内部都使用了锁，因此它们是线程安全的。实现时要尽可能保持简单，避免阻塞线程，不要再使用锁。