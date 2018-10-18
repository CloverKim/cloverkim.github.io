---
title: iOS开发 - 分享一个关于KVO的扩展
categories: iOS开发
tags:
- iOS
- kvo
- Swift
top: 110
copyright: ture
---

# 主要代码（Swift）
```
typealias KVONotificationBlock = (Any?, _ oldValue: Any?, _ value: Any?) -> Void

extension NSObject {
    //默认的函数，option的初始值是Initial|New, 监测打变化的值默认转到主线程
    func observe(_ object: Any?, keyPath: String, block: @escaping KVONotificationBlock) {
        self.kvoController.observe(object, keyPath: keyPath, options:[.initial, .new], block:{(observer: Any?, object: Any, change: [String: Any]) in
            DispatchQueue.main.async(execute: { () -> Void in
                block(observer, change["old"], change["new"]);
            })
        })
    }

    func observe(_ object: Any?, keyPath: String, options: NSKeyValueObservingOptions, mainThread: Bool, block: @escaping KVONotificationBlock) {
        self.kvoController.observe(object, keyPath: keyPath, options: options) { (observer, object, change: [String: Any]) -> Void in
            if !mainThread || Thread.isMainThread == true  {
                block(observer, change["old"], change["new"]);
            } else {
                DispatchQueue.main.async(execute: { () -> Void in
                    block(observer, change["old"], change["new"]);
                })
            }
        }
    }
    
    func removeObserve(_ object: Any?, path: String) {
        self.kvoController.unobserve(object, keyPath: path);
    }
    
    func removeObserve(_ obj: Any?) {
        self.kvoController.unobserve(obj);
    }
    
    func removeAll() {
        self.kvoController.unobserveAll();
    }
}
```
<!-- more -->
# 相关说明
## 使用的前提：
&emsp;&emsp;使用[CocoaPods](https://github.com/cocoapods/cocoapods)，在我们的工程项目的Podfile文件添加
```
pod 'KVOController'
```
## kvoController是啥？
&emsp;&emsp;[FBKVOController](https://github.com/facebook/KVOController)是Facebook开源的替代KVO的解决方案。它用block解决了以前使用KVO时代码散乱的缺点。
## 为啥要用FBKVOController？
&emsp;&emsp;kvo 全称 key-value observing，由 cocoa 框架提供的支持观察者模式的技术，结合 Objective-C 非常易用，在很多场合都可以有效地替换 NSNotificationCenter。但其也有一些致命的缺点，就是很容易导致引发 crash。譬如：只有addObserver，没有removeObserver。addObserver 和 removeObserver 必须配对出现，不然的话，等待你的就是crash。其调用的顺序：必须先添加观察者，然后处理业务，最后完成后移除观察者，释放掉观察者。removeObserver方法的调用，一般在dealloc()（OC）、deinit()（Swift）中。
&emsp;&emsp;而FBKVOController则是帮助我们更好的使用KVO。FBKVOController生命周期跟观察者绑定，则观察者释放时，由FBKVOController生成的实例也被释放，从 _FBKVOSharedController 移除对应的观察者信息，避免发消息给已释放观察者导致的crash。
## 为啥要写这个扩展？
&emsp;&emsp;在controller中直接调用observe方法，传入对应的参数，在block回调中做该做的事情就可以。当然，也不需要手动移除监听。
```
observe(scrollView, keyPath: #keyPath(UIScrollView.contentOffset)) { [weak self] (weakSelf, oldValue, newValue) in
    // 处理事情
}
```

