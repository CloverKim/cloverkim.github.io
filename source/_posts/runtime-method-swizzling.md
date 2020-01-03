---
title: iOS开发 - runtime实践·方法交换（Method Swizzling）
categories: iOS开发
tags:
- iOS
- runtime
- 实践
top: 100
copyright: ture
---

# 了解几个概念
1. Selector（typedef struct objc_selector *SEL）：在运行时Selectors用来代表一个方法的名字。Selector是一个在运行时被注册（或映射）的C类型字符串。Selector由编译期产生并且在当类被加载进内存时由运行时自动进行名字和实现的映射。
2. Method（typedef struct objc_method *Method）：是一个不透明的用来代表一个方法的定义的类型。<!-- more -->
3. Implementation（typedef id(*IMP)(id,SEL,...)）：这个数据类型指向一个方法的实现的最开始的地方。该方法为当前CPU架构使用标准的C方法调用来实现。该方法的第一个参数指向调用方法的自身（即内存中类的实例对象，若是调用类方法，该指针则是指向元类对象(metaclass)）；第二个参数是这个方法的名字Selector，该方法的真正参数紧随其后。

&emsp;&emsp;理解Selector，Method，Implementation这三个概念之间关系的最好方式是：在运行时，类(Class)维护了一个消息分发列表来解决消息的正确发送。每一个消息列表的入口是一个方法(Method)，这个方法映射了一对键值对，其中键值是这个方法的名字Selector(SEL)，值是指向这个方法实现的函数指针Implementation(IMP)。Method Swizzling修改了类的消息分发列表使得已经存在的Selector映射了另一个Implementation，同时重命名了原生方法的实现为一个新的Selector。

# Method Swizzling原理
&emsp;&emsp;Method Swizzling是发生在运行时的，主要用于在运行时将两个Method进行交换，Method Swizzling配合类别可以实现在不干扰其他工程代码的情况下与系统的方法进行交换。
![](http://pic.cloverkim.com/749c46aagy1fxmqwtqlguj20ft0b9aas.jpg)
&emsp;&emsp;在上图中，我们添加了selector3和IMP3，并让selector2指向了IMP3，而selector3则指向了IMP2，这样就实现了“方法交换”。
&emsp;&emsp;在Objective-C的runtime特性中，调用一个对象的方法就是给这个对象发送消息。是通过查找接收消息对象的方法列表，从方法列表中查找对应的SEL，这个SEL对应着一个IMP（一个IMP可以对应多个SEL），通过这个IMP找到对应的方法调用。
&emsp;&emsp;每个类中都有一个Dispatch Table，这个Dispatch Table本质是将类中的SEL和IMP进行对应。而我们的Method Swizzling就是对这个table进行了操作，让SEL对应另一个IMP。

# Method Swizzling用法
&emsp;&emsp;先给要交换的方法的类添加一个Category，然后在Category中的+(void)load（[iOS开发 - +load 和 +initialize 方法](https://cloverkim.com/method_load_initialize.html)） 方法中添加Method Swizzling方法，我们用来交换的方法也写在Category中。

## 两种使用方式：
- 第1种：

```
class_getInstanceMethod(Class _Nullable cls, SEL _Nonnull name)

method_getImplementation(Method _Nonnull m) 

class_addMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp, const char * _Nullable types) 

class_replaceMethod(Class _Nullable cls, SEL _Nonnull name, IMP _Nonnull imp, const char * _Nullable types) 
```

- 第2种：

```
method_exchangeImplementations(Method _Nonnull m1, Method _Nonnull m2) 
```

# 应用举例
- 打印当前进入的controller的名称。为了方便日常开发的调试，我们可以交换系统viewDidLoad方法的实现，在新的方法中，打印当前控制器的名称，方便我们定位当前页面对应的controller。当然，线上版本并不需要，因此要打上DEBUG。

```
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        
        SEL originalSelector = @selector(viewDidLoad);
        SEL swizzledSelector = @selector(newViewDidLoad);
        
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        
        BOOL didAddMethod = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
        
        if (didAddMethod) {
            class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(swizzledMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

- (void)newViewDidLoad {
    [self newViewDidLoad];
    
    NSLog(@"=================================================");
    NSLog(@"进入%@ ", NSStringFromClass([self class]));
    NSLog(@"=================================================");
}
```

- 防止popToViewController方法，找不到对应的controller时，导致crash。

```
+ (void)load {
    Method oriMethod = class_getInstanceMethod([UINavigationController class], @selector(popToViewController:animated:));
    Method exchangeMethod = class_getInstanceMethod([UINavigationController self], @selector(runtime_popToViewController:animated:));
    method_exchangeImplementations(oriMethod, exchangeMethod);
}

- (NSArray<UIViewController *> *)runtime_popToViewController:(UIViewController *)controller animated:(BOOL)animated {
    if ([self.viewControllers containsObject:controller]) {
        return [self runtime_popToViewController:controller animated:animated];
    } else {
        UIViewController *parent = controller.parentViewController;
        while (parent != nil) {
            if ([self.viewControllers containsObject:parent]) {
                return [self runtime_popToViewController:parent animated:animated];
            }
            parent = parent.parentViewController;
        }
        NSAssert(YES, @"navigation's controllers doesn't contain this controller");
        return @[];
    }
}
```

- 防止因为数组越界而导致的crash（不推荐，有问题）
&emsp;&emsp;在iOS中NSNumber、NSArray、NSDictionary等这些类都是类簇（Class Clusters），一个NSArray的实现可能由多个类组成。所以如果想对NSArray进行Swizzling，必须获取到其“真身”进行Swizzling，直接对NSArray进行操作是无效的。这是因为Method Swizzling对NSArray这些的类簇是不起作用的。
&emsp;&emsp;因为这些类簇类，其实是一种抽象工厂的设计模式。抽象工厂内部有很多其他继承自当前类的子类，抽象工厂类会根据不同情况，创建不同的抽象对象来进行使用。例如我们调用NSArray的objectAtIndex：方法，这个类会在方法内部判断，内部创建不同抽象类进行操作。
&emsp;&emsp;所以如果我们对NSArray类进行Swizzling操作其实只是对父类进行了操作，在NSArray内部会创建其他子类来执行操作，真正执行Swizzling操作的并不是NSArray自身，所以我们应该对其“真身”进行操作。
&emsp;&emsp;下面列举了NSArray和NSDictionary本类的类名，可以通过Runtime函数取出本类：

| 类名 | 真身 |
| --- | --- |
| NSArray | __NSArrayI |
| NSMutableArray  | __NSArrayM |
| NSDictionary | __NSDictionaryI |
| NSMutableDictionary | __NSDictionaryM |

```
+ (void)load {
    Method oriMethod = class_getInstanceMethod(object_getClass(@"__NSArrayI"), @selector(objectAtIndex:));
    Method exchangeMethod = class_getInstanceMethod(object_getClass(@"__NSArrayI"), @selector(runtime_objectAtIndex:));
    method_exchangeImplementations(oriMethod, exchangeMethod);
}

- (id)runtime_objectAtIndex:(NSUInteger)index {
    if (self.count - 1 < index) {
        NSAssert(YES, @"Array indices are out of bounds");
        return nil;
    } else {
        return [self runtime_objectAtIndex:index];
    }
}
```
**这里需要注意：**
- 当创建的数组，为空时，其class为__NSArray0，当数组只有一个元素时，其class为__NSSingleObjectArrayI，其他情况下，其class才为__NSArrayI。因此使用上述的方法进行防止crash时，需要考虑使用。因此不推荐该方法。
- 为何在交换的方法中，调用该方法，为何不调用原交换方法？因为系统的方法已经被我们交换了，因此我们写的交换方法的SEL指向的其实是系统原来的方法，如果我们再调用原交换方法的话，就会造成死循环了。这里需要注意！

# 参考
- [简书 - 陈满iOS - iOS开发·runtime原理与实践: 方法交换篇(Method Swizzling)(iOS“黑魔法”，埋点统计，禁止UI控件连续点击，防奔溃处理)](https://www.jianshu.com/p/6bcff1f9feee)
- [CocoaChina - 枣泥布丁 - iOS开发之 Method Swizzling 深入浅出](http://www.cocoachina.com/ios/20180426/23192.html)
