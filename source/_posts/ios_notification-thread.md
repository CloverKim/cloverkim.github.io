---
title: iOS开发 - NSNotification和线程相关
categories: iOS开发
tags:
- iOS
- 基础
top: 100
copyright: ture
---

# 问题的开始
&emsp;&emsp;我们都知道NSNotification是线程同步的，但有时却很容易忽视线程同步这个特性带来的问题。比如举下面的例子：<!-- more -->
```
- (IBAction)notificationAction {
    NSLog(@"开始发通知");
    [[NSNotificationCenter defaultCenter] postNotificationName:@"knotificationDemo" object:nil];
    NSLog(@"通知结束了");
}
```
&emsp;&emsp;然后在接收通知的controller中模拟一个耗时的操作：
```
- (void)notify {
    NSLog(@"模拟耗时操作");
    [NSThread sleepForTimeInterval:3.0];
    NSLog(@"耗时操作结束");
}
```

&emsp;&emsp;控制台的打印结果如下图所示：
![](https://ws1.sinaimg.cn/large/006tNc79gy1fzoh957aghj30fi03ldgz.jpg)
&emsp;&emsp;看到执行结果的打印，我们就能大致理解Notification的线程同步的特性了。在主线程中发出通知，然后接收方在主线程处理逻辑，并接收方处理完毕时，发送方才能继续执行剩下的逻辑。
&emsp;&emsp;那么，Notification和线程同步之间到底是什么关系呢?
&emsp;&emsp;官方文档说明如下：
> In a multithreaded application, notifications are always delivered in the thread in which the notification was posted, which may not be the same thread in which an observer registered itself.
> 
> 在多线程应用程序中，通知总是在发出通知的线程中传递，而该线程不一定是观察者观察者的那个线程。

# 重定向
&emsp;&emsp;那如果我们的Notification是在其他线程中post的，如何能在主线程中对这个Notification进行处理呢？或者说：如果我们希望一个Notification的post线程与接收线程不是同一个线程，应该怎么办？先来看下官方文档的相关说明：
> For example, if an object running in a background thread is listening for notifications from the user interface, such as a window closing, you would like to receive the notifications in the background thread instead of the main thread. In these cases, you must capture the notifications as they are delivered on the default thread and redirect them to the appropriate thread.

> 例如，如果在后台线程中运行的对象正在监听来自用户界面的通知，例如窗口关闭，则希望在后台线程而不是主线程中接收通知。在这些情况下，您必须在默认线程上传递通知时捕获它们，并将它们重定向到适当的线程中。

&emsp;&emsp;官方文档中讲到了“重定向”，就是我们在Notification所在的默认线程中捕获这些分发的通知，然后将其重定向到指定的线程中。
&emsp;&emsp;一种重定向的实现思路是自定义一个通知队列（**注意，不是NSNotificationQuene对象，而是一个数组**），让这个队列去维护那些我们需要重定向的Notification。我们仍然是像平常一样去注册一个通知的观察者，当Notification来了时，先看看post这个Notification的线程是不是我们所期望的线程，如果不是，则将这个Notification存储到我们的队列中，并发送一个信号（Signal）到期望的线程中，来告诉这个线程需要处理一个Notification。指定的线程在收到信号后，将Notification从列表中移除，并进行处理。
&emsp;&emsp;下面借助官方文档给出的demo，进行测试看下实际结果：
```
@interface ViewController ()<NSMachPortDelegate>

@property(nonatomic) NSMutableArray     *notifications;         // 通知队列
@property(nonatomic) NSThread           *notificationThread;    // 期望线程
@property(nonatomic) NSLock             *notificationLock;      // 用于对通知队列加锁的锁对象，避免线程冲突
@property(nonatomic) NSMachPort         *notificationPort;      // 用于向期望线程发送信号的通信端口

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view, typically from a nib.
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(notify) name:@"knotificationDemo" object:nil];
    
    NSLog(@"...");
    
    NSLog(@"current thread = %@", [NSThread currentThread]);
    
    // 初始化
    self.notifications = [[NSMutableArray alloc] init];
    self.notificationLock = [[NSLock alloc] init];
    self.notificationThread = [NSThread currentThread];
    self.notificationPort = [[NSMachPort alloc] init];
    self.notificationPort.delegate = self;
    
    // 往当前线程的runloop添加端口源
    // 当Mach消息到达而接收线程的runloop没有运行时，则内核会保存这条消息，直到下一次进入runloop
    [[NSRunLoop currentRunLoop] addPort:self.notificationPort forMode:(__bridge NSString *)kCFRunLoopCommonModes];
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(processNotification:) name:@"ktestNotification" object:nil];
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:@"ktestNotification" object:nil userInfo:nil];
    });
    
}

- (void)handleMachMessage:(void *)msg {
    [self.notificationLock lock];
    
    while ([self.notifications count]) {
        NSNotification *notification = [self.notifications objectAtIndex:0];
        [self.notifications removeObjectAtIndex:0];
        [self.notificationLock unlock];
        [self processNotification:notification];
        [self.notificationLock lock];
    };
    
    [self.notificationLock unlock];
}

- (void)processNotification:(NSNotification *)notification {
    if ([NSThread currentThread] != self.notificationThread) {
        // 将通知转发到正确的线程
        [self.notificationLock lock];
        [self.notifications addObject:notification];
        [self.notificationLock unlock];
        [self.notificationPort sendBeforeDate:[NSDate date]
                                   components:nil
                                         from:nil
                                     reserved:0];
    } else {
        // 在这里处理通知
        NSLog(@"current thread = %@", [NSThread currentThread]);
        NSLog(@"process notification");
    }
}
```

&emsp;&emsp;运行后的输出结果如下：
![](https://ws2.sinaimg.cn/large/006tNc79gy1fzop4fxlmwj30ip03sjsk.jpg)

&emsp;&emsp;由上图的运行结果可以看出，我们在全局dispatch队列中抛出的Notification，如愿的在主线程中接收到了。
&emsp;&emsp;这种实现方式的具体解析以及其局限性大家可以参考官方文档[Delivering Notifications To Particular Threads](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Notifications/Articles/Threading.html#//apple_ref/doc/uid/20001289-CEGJFDFG)。当然，更好的方法可能是我们自己去子类化一个NSNotificationCenter，或者单独写一个类来处理这种转发。
&emsp;&emsp;而且官方文档告诉我们，NSNotificationCenter是一个线程安全类，我们可以在多线程环境下使用同一个NSNotificationCenter对象而不需要加锁。原文在[Threading Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/ThreadSafetySummary/ThreadSafetySummary.html)。

# 参考
- [Apple官方文档 - Notification Programming Topics](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Notifications/Articles/Notifications.html)
- [简书 - 高浩浩浩浩浩浩 - NSNotification 的细节](https://www.jianshu.com/p/4f47a8f07f0e)
- [Cocoa China - lansekuangtu - Notification与多线程](http://www.cocoachina.com/ios/20150316/11335.html)
- [Observers and Thread Safety](http://inessential.com/2013/12/20/observers_and_thread_safety)