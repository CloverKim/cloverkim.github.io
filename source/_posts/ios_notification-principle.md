---
title: iOS开发 - NSNotification原理理解
categories: iOS开发
tags:
- iOS
- 基础
top: 100
copyright: ture
---

# 概念
&emsp;&emsp;NSNotification是iOS中一个调度消息通知的类，采用单例设计模式，在开发中实现传值、回调等。在iOS中，NSNotification是使用观察者模式来实现用于跨层传递消息。

<!-- more -->

# 三个主要类
## NSNotification
&emsp;&emsp;NSNotification包含了一些用于向其他对象发送通知的必要信息，包括名称、对象和可选字典，并由NSNotificationCenter或NSDistributedNotificationCenter的实例进行发送。name是标识通知的标记、object是保存发送通知的对象、userinfo存储其他相关对象。**这里主要注意的是：NSNotification对象是不可变的。**

| 字段名 | 含义 |
| :-: | :-: |
| name | 通知的名称，用于通知的唯一标识 |
| object | 保存发送通知的对象 |
| userinfo | 存储其他相关对象 |

&emsp;&emsp;可以使用` notificationWithName:object: ` 或 ` notificationWithName:object:userInfo: ` 创建通知对象。但实际开发中，一般是直接使用NSNotificationCenter调用 ` postNotificationName:object: ` 或 ` postNotificationName:object:userInfo: ` ，这两个类方法会在内部直接创建NSNotification对象，并发出通知。
&emsp;&emsp;从官网文档可知，NSNotification是不能直接实例化的，如果用init方法进行实例化时，会引发异常。还有需要注意的是如果我们自己去实现构造方法时，不能在super上调用init方法。

## NSNotificationCenter
&emsp;&emsp;NSNotificationCenter提供了一套机制来发送通知，每个运行中的应用程序都有一个defaultCenter通知中心，我们可以创建新的通知中心来组织特定上下文中的通信。&emsp;&emsp;NSNotificationCenter暴露给外部的字段只有一个defaultCenter，并且该字段是只读的，暴露出来的方法分为三种：添加、移除通知观察者和发出通知。详细如下表所示：


| 作用 | 相关方法 |
| :-: | :-: |
| 添加通知观察者 | addObserver:selector:name:object:<br>addObserverForName:object:queue:usingBlock: |
| 移除通知观察者 | removeObserver: <br>removeObserver:name:object: |
| 发出通知 | postNotification:<br>postNotificationName:object: <br>postNotificationName:object:userInfo: |

&emsp;&emsp;相关说明：
- ` addObserverForName:object:queue:usingBlock: `方法，相比` addObserver:selector:name:object: `方法多了queue和block，queue就是决定将block回调提交到哪个队列里面执行。这里需要注意的是发送通知和接收通知的线程必须为同一个。常见情况下会把queue设置为主队列，因为主队列的任务都会在主线程下完成，因此可以用这种方式来实现通知更新UI。

## NSNotificationQueue
&emsp;&emsp;简单理解为：通知中心的缓冲区。尽管通知中心已经分发通知，但放置到队列中的通知可能会延迟，直到runloop结束或者runloop空闲时才发送。如果有多个相同的通知，NSNotificationQueue会将其进行合并，以便在发布多个通知的情况下只发送一个通知。
&emsp;&emsp;通知队列按照先进先出(FIFO)的顺序维护通知。当一个通知移动到队列的前面时，队列将它发送到通知中心，然后再将通知分派给所有注册为观察者的对象。每个线程都有一个默认的通知队列，该队列与流程的默认通知中心相关联。我们也可以创建自己的通知队列。
&emsp;&emsp;和NSNotificationCenter一样，NSNotificationQueue也只暴露了一个字段：defaultQueue，返回当前线程的默认通知队列。方法分为：创建通知队列和管理通知。详细说明如下表所示：


| 作用 | 相关方法 |
| :-: | :-: |
| 创建通知队列 | initWithNotificationCenter: |
| 管理通知 | enqueueNotification:postingStyle:<br> dequeueNotificationsMatching:coalesceMask:<br>enqueueNotification:postingStyle:coalesceMask:forModes: |

&emsp;&emsp;方法相关说明：
- ` initWithNotificationCenter: ` 初始化并返回指定通知中心的通知队列。
- ` enqueueNotification:postingStyle: ` 使用指定的发布样式向通知队列添加通知。
- ` dequeueNotificationsMatching:coalesceMask: ` 使用提供的匹配条件从匹配提供的通知的队列中删除所有通知。
- ` enqueueNotification:postingStyle:coalesceMask:forModes: ` 使用指定的发布样式、合并标准和运行循环模式向通知队列添加通知。

&emsp;&emsp;在上面的方法中，需要注意的2个常量，相关说明如下：
- NSPostingStyle：用于指定何时发布通知
 * NSPostASAP：在当前通知调用或者计时器结束时发出通知
 * NSPostWhenIdle：当runloop处于空闲时发出通知
 * NSPostNow：当合并通知完成之后立即发出通知

- NSNotificationCoalescing：用于指定通知如何合并
 * NSNotificationNoCoalescing：不合并通知
 * NSNotificationCoalescingOnName：合并具有相同名称的通知
 * NSNotificationCoalescingOnSender： 将通知与相同的对象合并

# 实现原理
&emsp;&emsp;NSNotificationCenter定义了两个Table，同时为了封装观察者信息，也定义了Observation保存观察者信息。他们的结构体可以简化如下所示：
```
typedef struct NCTbl {
    Observation   *wildcard;  // 保存既没有没有传入通知名字也没有传入object的通知
    MapTable       nameless;  // 保存没有传入通知名字的通知
    MapTable       named;     // 保存传入了通知名字的通知
} NCTable;
```
```
typedef struct Obs {
    id        observer;       // 保存接受消息的对象
    SEL       selector;       // 保存注册通知时传入的SEL
    struct Obs    *next;      // 保存注册了同一个通知的下一个观察者
    struct NCTbl  *link;      // 保存改Observation的Table
} Observation;
```
&emsp;&emsp;在NSNotificationCenter内部一共保存了两张表，一张用于保存添加观察者的时候传入的NotificationName的情况；一张用于保存添加观察者的时候没有传入NotificationCenter的情况，详细分析如下：
## Table
### Named Table
&emsp;&emsp;在Named Table中，NotificationName作为表的key，因为我们在注册观察者的时候是可以传入一个object参数用于只监听该对象发出的通知，并且一个通知可以添加多个观察者，所以还需要一张表用来保存object和observe的对应关系。这张表的key、value分别是以object为key，observe为value。所以对于Named Table，最终的结构为：
- 首先外层有一个Table，以通知名称为key。其value同样是一个Table。
- 为了实现可以传入一个参数object用于只监听指定该对象发出的通知，及一个通知可以添加多个观察者。则内Table以传入的object为key，用链表来保存所有的观察者，并且以这个链表为value。

![](http://pz1livcqe.bkt.clouddn.com/Named Table.jpg 'Named Table')


&emsp;&emsp;特别说明：在实际开发中，我们经常将object参数传nil，这个时候系统会根据nil自动产生一个key。相当于这个key对应的value（链表）保存的就是对于当前NotificationName没有传入object的所有观察者。当NotificationName被发送时，在链表中的观察者都会收到通知。

### UnNamed Table
&emsp;&emsp;UNamed Table结构比Named Table简单得多。因为没有NotificationName作为key。这里直接就以object为key，比Named Table少了一层Table嵌套。
![](http://pz1livcqe.bkt.clouddn.com/UnNamed Table.jpg 'UnNamed Table')

&emsp;&emsp;如果在注册观察者时没有传入NotificationName，同时没有传入object，所有的系统通知都会发送到注册的对象里。

## 添加观察者的流程
&emsp;&emsp;首先在初始化NSNotificationCenter时会创建一个对象，这个对象里面保存了Named Table、UNamed Table和其他信息。
1. 首先会根据传入的参数，实例化一个Observation。该Observation对象保存了观察者对象、接收到通知观察者对所执行的方法，由于Observation是一个链表，还保存了下一个Observation的地址。
2. 根据是否传入通知的Name，选择在Named Table还是UNamed Table进行操作。
3. 如果传入通知的name，则会先去用name去查找是否已经有对应的value（注意这个时候返回的value是一个Table）
4. 如果没有对应的value，则创建一个新的Table，然后将这个Table以name为key添加到Named Table。如果有value，那直接去取出这个Table。
5. 得到了保存Observation的Table之后，就通过传入的object拿对应的链表。如果object为空，会默认有一个key表示传入object为空的情况，取的时候也会直接用这个key去取，表示任何地方发送通知都会监听。
6. 如果保存Observation的Table中根据object作为key没有找到对应的链表时，则会创建一个节点，作为头结点插入进去；如果找到了则直接在链表末尾插入之前实例化好的Observation中。

&emsp;&emsp;在没有传入NotificationName的情况和上面的过程类似，只不过是直接根据object去对应的链表而已。如果既没有传入NotificationName，也没有传入object，则这个观察者会添加到wildcard链表中。

## 发送通知的流程
&emsp;&emsp;发送通知一般是调用` postNotificationName:object:userInfo: `方法来实现。该方法内部会实例化一个NSNotification来保存传入的各种参数，包括name、object和userinfo。
&emsp;&emsp;发送通知的流程总体来说就是根据NotificationName查找到对应的Observer链表，然后遍历整个链表，给每个Observer结点中保存的对象及SEL，来向对象发送消息。具体流程如下：
1. 首先会定义一个数组ObserversArray来保存需要通知的Observer。之前在添加观察者的时候把既没有传入NotificationName，也没有传入object的，保存在了wildcard。因为这样观察者会监听所有NotificationName的通知，所以先把wildcard链表遍历一遍，将其中的Observer加到数组ObserversArray中。
2. 找到以object为key的Observer链表。这个过程分为：在Named Table中查找，以及在UNamed Table中查找。然后将遍历找到的链表，同样加入到最开始创建的数组ObserversArray中。
3. 至此，所有关于NotificationName的Observer（wildcard + UNamed Table + Named Table）已经加入到了数组ObserversArray中。接下来就是遍历这个ObserversArray数组，一次取出其中的Observer结点。因为这个结点保存了观察者对象以及selector。所以最终调用形式如下：

    ```
     [observerNode->observer performSelector: o->selector withObject: notification];
    ```
&emsp;&emsp;这个方式也就能说明，发送通知的线程和接收通知的线程都是同一个线程。

# NSNotification与多线程
&emsp;&emsp;NSNotification和线程同步之间是什么关系呢？先看下官方文档的说明：
> In a multithreaded application, notifications are always delivered in the thread in which the notification was posted, which may not be the same thread in which an observer registered itself.

翻译过来意思为：
> 在多线程应用程序中，通知总是在发出通知的线程中传递，而该线程不一定是观察者观察者的那个线程。

&emsp;&emsp;更多关于NSNotification与线程之间的关系，请阅读下面的文章：[iOS开发 - NSNotification和线程相关](http://cloverkim.com/ios_notification-thread.html)

# 总结
&emsp;&emsp;总的来说，NSNotification的三个相关类的作用，可以用下图进行归纳总结。
![](http://pz1livcqe.bkt.clouddn.com/总结.jpg '总结')

# 参考
- [Apple官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Notifications/Articles/Notifications.html)
- [简书 - 纸简书生 - 深入理解iOS NSNotification](https://www.jianshu.com/p/83770200d476)
- [CSDN - YFL_iOS - iOS Notification实现原理](https://blog.csdn.net/qq_18505715/article/details/76146575)
- [简书 - 高浩浩浩浩浩浩 - NSNotification 的细节](https://www.jianshu.com/p/4f47a8f07f0e)