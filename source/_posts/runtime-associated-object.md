---
title: iOS开发 - runtime实践·关联对象（Associated Object）
categories: iOS开发
tags:
- iOS
- runtime
- 实践
top: 100
copyright: ture
---

# 什么是关联对象
&emsp;&emsp;关联对象是指某个对象通过一个唯一的key连接到一个类的实例上。我们都知道，可以使用Category来扩展方法，那Category可以添加属性吗？相信很多人都会回答不可以，答案其实是可以的，只是不会自动生成getter/setter方法的实现，也不会自动生成成员变量，在我们调用的时候就会crash了。如果要添加一个属性，那就要用到runtime的关联对象了。
<!-- more -->
# 如何关联对象
&emsp;&emsp;来看下runtime提供给我们的3个API方法：
```
// 关联对象
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
// 获取关联的对象
id objc_getAssociatedObject(id object, const void *key)
// 移除关联的对象
void objc_removeAssociatedObjects(id object)
```
相关说明：
- id object：被关联的对象
- const void *key：关联的key，要求唯一
- id value：关联的对象
- objc_AssociationPolicy policy：内存管理的策略

objc_AssociationPolicy的枚举值和相关说明：
```
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,            // 指定一个弱引用相关联的对象
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1,  // 指定相关对象的强引用，非原子性
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,    // 指定相关的对象被复制，非原子性
    OBJC_ASSOCIATION_RETAIN = 01401,        // 指定相关对象的强引用，原子性
    OBJC_ASSOCIATION_COPY = 01403           // 指定相关的对象被复制，原子性   
};
```
&emsp;&emsp;当对象被释放时，会根据这个策略会决定是否释放关联的对象，当策略是RETAIN/COPY时，会释放关联的对象，当是ASSIGN时，将不会释放。需要注意的是，我们不需要主动调用removeAssociated方法来解除关联的对象，如果需要解除指定的对象，可以使用setAssociatedObject置nil来实现。

使用步骤：
- 创建对应类的一个Category，创建要添加的属性。
- 重写属性的setter方法，在setter方法中调用objc_setAssociatedObject进行关联对象。
- 重写属性的getter方法，在getter方法中return objc_getAssociatedObject（获取到的关联对象）
- 需要注意的是：key一定要唯一，可以定义成宏。

# 应用场景
1. 给UITapGestureRecognizer动态添加一个NSString类型的属性，方便传递字符串类型的信息，当然UITapGestureRecognizer本身也有tag。

&emsp;&emsp;在.h文件中进行声明
```
NS_ASSUME_NONNULL_BEGIN

@interface UITapGestureRecognizer (NSString)

@property (nonatomic, strong) NSString *tapString;

@end

NS_ASSUME_NONNULL_END
```
&emsp;&emsp;在.m文件中
```
static NSString *tapStringKey = @"tapString";

@implementation UITapGestureRecognizer (NSString)

- (void)setTapString:(NSString *)tapString {
    objc_setAssociatedObject(self, &tapStringKey, tapString, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
}

- (NSString *)tapString {
    return objc_getAssociatedObject(self, &tapStringKey);
}
```
&emsp;&emsp;在viewController中就可以调用了：
```
UITapGestureRecognizer *tap = [[UITapGestureRecognizer alloc] initWithTarget:self action:@selector(tapAction:)];
tap.tapString = @"触摸了！";
[self.view addGestureRecognizer:tap];

- (void)tapAction:(UITapGestureRecognizer *)tap {
    NSLog(@"%@", tap.tapString);
}
```

2. 假如封装好了一个页面缺省图，我们定义为BlankPageView，如果我们要为每个UIView添加一个blankPageView属性时，可以使用关联对象的方法。

&emsp;&emsp;在setter方法中：
```
static char BlankPageViewKey;

- (void)setBlankPageView:(BlankPageView *)blankPageView {
    objc_setAssociatedObject(self, &BlankPageViewKey,
                                blankPageView,
                                OBJC_ASSOCIATION_RETAIN_NONATOMIC);               
}
```
&emsp;&emsp;在getter方法中：
```
- (BlankPageView *)blankPageView {
    return objc_getAssociatedObject(self, &BlankPageViewKey);
}
```

3. 以UIButton为例，使用关联对象完成一个函数的点击回调

&emsp;&emsp;在分类的.h文件中
```
typedef void(^BtnActionCallBack)(UIButton *btn);

@interface UIButton (ActionBlock)

- (void)btnActionCallBack:(BtnActionCallBack)callBack;

@end
```
&emsp;&emsp;在.m文件中
```
static NSString *btnActionKey = @"btnAction";

- (void)btnActionCallBack:(BtnActionCallBack)callBack {
    objc_setAssociatedObject(self, &btnActionKey, callBack, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    
    [self addTarget:self action:@selector(btnAction) forControlEvents:UIControlEventTouchUpInside];
}

- (void)btnAction {
    BtnActionCallBack callBack = objc_getAssociatedObject(self, &btnActionKey);
    
    if (callBack) {
        callBack(self);
    }
}
```
&emsp;&emsp;使用：
```
[self.testBtn btnActionCallBack:^(UIButton * _Nonnull btn) {
    NSLog(@"button action call back");
}];
```

4. 关联观察者对象
&emsp;&emsp;当在一个Category的实现中使用KVO时，建议用一个自定义的关联对象而不是该对象本身作为观察者。比如AFNetworking，为loading控件监听NSURLSessionTask以获取网络进度的分类中：
```
@implementation UIActivityIndicatorView (AFNetworking)
- (AFActivityIndicatorViewNotificationObserver *)af_notificationObserver {
    
    AFActivityIndicatorViewNotificationObserver *notificationObserver = objc_getAssociatedObject(self, @selector(af_notificationObserver));
    if (notificationObserver == nil) {
        notificationObserver = [[AFActivityIndicatorViewNotificationObserver alloc] initWithActivityIndicatorView:self];
        objc_setAssociatedObject(self, @selector(af_notificationObserver), notificationObserver, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    }
    return notificationObserver;
}
- (void)setAnimatingWithStateOfTask:(NSURLSessionTask *)task {
    [[self af_notificationObserver] setAnimatingWithStateOfTask:task];
}
@end
```

# 参考
- [简书 - 陈满iOS - iOS开发·runtime原理与实践: 关联对象篇(Associated Object)(应用场景：为分类添加“属性”，为UI控件关联事件Block体，为了不重复获得某种数据)](https://www.jianshu.com/p/916aef6f7ab1)
- [简书 - 明仔Su - iOS runtime实战应用：关联对象](https://www.jianshu.com/p/c68cc81ef763)
