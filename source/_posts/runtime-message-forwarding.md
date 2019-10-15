---
title: iOS开发 - runtime实践·消息转发（Message Forwarding）
categories: iOS开发
tags:
- iOS
- runtime
- 实践
top: 100
copyright: ture
---

# 消息传递机制
&emsp;&emsp;在OC中，当调用了某个对象的方法时，其实质上就是向该对象发送了一条消息，OC的方法最终被生成了C函数，并附带额外的参数。消息传递机制中所调用的核心函数为：
```
objc_msgSend(id _Nullable self, SEL _Nonnull op, ...)
```
<!-- more -->
&emsp;&emsp;该函数是个参数可变的函数，能接收两个及以上的参数，第一个参数为方法接收者，第二个参数为选择器，后续的参数则为方法调用所需要的相应参数。
&emsp;&emsp;举个🌰：
```
[array insertObject:message atIndex: 3];
```
&emsp;&emsp;在编译时，我们上面所写的OC函数会被转化成如下C函数：
```
objc_msgSend(array, @selector(insertObject:atIndex:), message, 3);
```

# 消息转发
&emsp;&emsp;当向某个对象发送一条消息时，若该对象的方法列表以及它相应继承链上的方法列表都无法找到以该消息选择子作为key的方法实现时，则会触发消息转发机制。
&emsp;&emsp;如果没有方法的实现，程序会在运行时crash并抛出**unrecognized selector sent to instance**的异常，但在抛出异常之前，OC的runtime会给我们3次拯救程序的机会。
1. 动态方法解析

```
+ (BOOL)resolveInstanceMethod:(SEL)sel;
```
&emsp;&emsp;当接收到未能识别的选择子时，运行时系统会调用该函数用以给对象一次机会来添加相应的方法实现，如果用户在该函数中动态添加了相应方法的实现，则跳转到方法的实现部分，并将该实现存入缓存中，以供下次调用。举个🌰：
```
- (void) btnActionMethod {
    NSLog(@"btnAction");
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(btnAction)) {
        class_addMethod([self class], sel, class_getMethodImplementation([self class], @selector(btnActionMethod)),  "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```
&emsp;&emsp;关于class_addMethod方法const char *types参数：
- "v@:"：这是一个void类型的方法，没有参数传入
- "i@:"：这是一个int类型的方法，没有参数传入
- "i@:@"：这是一个int类型的方法，有一个参数传入
- 更多相关，可以参考官方文档说明：[Type Encodings](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)

2. 备援接收者

```
- (id)forwardingTargetForSelector:(SEL)aSelector;
```
&emsp;&emsp;如果运行时在消息转发的第一步中未找到所调用方法的实现，那么当前接收者还有第二次机会进行未知选择子的处理。这是运行时系统会调用上述方法，并将未知选择子作为参数传入，该方法可以返回一个能处理该选择子的对象，运行时系统会根据返回的对象进行查找，若找到则跳转到相应的方法的实现，则消息转发结束。

3. 完整的消息转发

```
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```
&emsp;&emsp;当运行时系统检测到第二步中用户未返回能处理相应选择子的对象时，那么来到这一步就要启动完成的消息转发机制了。该方法可以改变消息调用目标，运行时系统根据所改变的调用目标，向调用目标方法列表中查询对应方法的实现并实现跳转，这种方式和第二步的操作非常相似。当然，你也可以修改方法的选择子，亦或者向所调用方法中追加一个参数等来跳转到相关方法的实现。
&emsp;&emsp;最后，如果消息转发的第三步还未能处理未知选择子的话，那么最终会调用NSObject类的如下方法，用以异常的抛出，表明该选择子最终未能处理。
```
- (void)doesNotRecognizeSelector:(SEL)aSelector;
```
&emsp;&emsp;对于完整的消息转发流程图如下图所示：
![](http://pz1livcqe.bkt.clouddn.com/消息转发流程图.jpg '消息转发流程图')

# 消息转发实例&验证
&emsp;&emsp;在新建的Project中，添加Cat、Dog、Rabbit三个类，并在每个类的.h文件中声明jump方法。
1. 我们用Cat类来验证消息转发的第一步方法：resolveInstanceMethod:，在该方法中动态添加jump方法的实现，使用runtime的class_addMethod方法，该方法用以向该类的实例对象中添加相应的方法实现。

```
void jump(id self, SEL cmd) {
    NSLog(@"%@  jump", self);
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if ([NSStringFromSelector(sel) isEqualToString:@"jump"]) {
        class_addMethod(self, sel, (IMP)jump, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```
&emsp;&emsp;然后在ViewController中创建Cat对象，并调用jump方法，如果Cat类中没有实现jump方法，正常的情况下，是会crash，但我们在resolveInstanceMethod: 方法中动态添加了方法的实现，则会跳转到新的方法实现中，因此才不会导致crash。控制台的打印如下：
![](http://pz1livcqe.bkt.clouddn.com/动态方法解析.jpg '动态方法解析')

2. 我们用Dog类来验证消息转发的第二步过程，根据上面所述的流程图中，为了能让运行时系统能够运行到forwardingTargetForSelector: 方法，我们需要在resolveInstanceMethod: 方法中返回NO，并且在forwardingTargetForSelector: 方法中返回Cat类的实例对象，让Cat类的实例对象去处理Dog的jump方法。

```
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    return NO;
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    if ([NSStringFromSelector(aSelector) isEqualToString:@"jump"]) {
        return [[Cat alloc] init];
    }
    return [super forwardingTargetForSelector:aSelector];
}
```
&emsp;&emsp;在ViewController中创建Dog对象，并调用jump方法。由于我们在forwardingTargetForSelector: 方法中，返回了Cat的实例对象，因此Dog的jump方法转发给了能处理Dog jump方法的Cat对象，并跳转到Cat对象的jump方法中，因此也不会导致crash。控制台的打印如下：
![](http://pz1livcqe.bkt.clouddn.com/备援接收者.jpg '备援接收者')

3. 最后用Rabbit类来验证消息转发的第三个步骤的过程。为了能触发完整的消息转发，我们需要将resolveInstanceMethod: 方法返回NO，并且在forwardingTargetForSelector: 方法中返回nil。另外，在实现forwardInvocation: 方法时，还需要实现methodSignatureForSelector: 方法，并将相应的选择子的描述返回。

```
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    return NO;
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    return nil;
}

- (void)forwardInvocation:(NSInvocation *)anInvocation {
    [anInvocation invokeWithTarget:[[Cat alloc] init]];
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    if ([NSStringFromSelector(aSelector) isEqualToString:@"jump"]) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector:aSelector];
}
```
&emsp;&emsp;最后在ViewController中创建Rabbit的实例，并调用jump方法，由于我们通过完整的消息转发，将方法的实现跳转到了Cat实例中，因此不但不会crash，还会执行Cat的jump方法，控制台的打印如下：
![](http://pz1livcqe.bkt.clouddn.com/完整的消息转发.jpg '完整的消息转发')


# 引申 - 向一个nil对象发送消息会怎样？
&emsp;&emsp;结论：OC中向为nil的对象发送消息，程序是不会crash的。
&emsp;&emsp;因为OC的函数都是通过```objc_msgSend```进行消息发送来实现，相对于C和C++来说，对于空指针的操作会引起crash问题，而```objc_msgSend```会通过判断self来决定是否发送消息，如果self为nil，那么selector也会为空，直接返回，不会出现问题。视方法返回值，向nil发消息可能会返回nil（返回值为对象），0（返回值为一些基础数据）或0X0（返回值为id）等。但对于```[NSNull null]```对象发送消息时，是会crash的，因为NSNull类只有一个null方法。

# 参考
- [博客园 - 俊华的博客 - Runtime 运行时之一：消息转发](https://www.cnblogs.com/junhuawang/p/5196291.html)
- [简书 - 陈满iOS - iOS开发·runtime原理与实践: 消息转发篇(Message Forwarding) (消息机制，方法未实现+API不兼容奔溃，模拟多继承)](https://www.jianshu.com/p/2fd4b930588e)
- [简书 - 高浩浩浩浩浩浩 - Objc中向一个nil对象发送消息会怎样](https://www.jianshu.com/p/11dca953f962)
- [Objective-C Runtime Programming Guide](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)