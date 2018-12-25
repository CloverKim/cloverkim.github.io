---
title: iOS开发 - KVO原理分析与使用
categories: iOS开发
tags:
- iOS
- 基础
- 观察者设计模式
top: 100
copyright: ture
---

# 概念
&emsp;&emsp;KVO全称KeyValueObserving，是Objective-C对观察者设计模式的一种实现。允许对象监听另一个对象特定属性的改变，并在改变时接收到事件。由于KVO的实现机制，所以对属性才会好发生作用，一般继承自NSObject的对象都默认支持KVO。

<!-- more -->

# 实现原理
&emsp;&emsp;当观察某对象A时，KVO机制动态创建一个对象A的子类，并为这个新的子类重写了被观察属性keyPath的setter方法。setter方法随后负责通知观察对象属性的改变。
&emsp;&emsp;KVO使用了isa-swizzling来实现。当观察对象A时，KVO机制动态创建一个新的类为：NSKVONotifying_A，该类继承自对象A的本类，且KVO为NSKVONotifying_A重写观察属性的setter方法，setter方法会负责在调用原setter方法之前和之后，通知所有观察对象属性值的更改情况。
&emsp;&emsp;NSKVONotifying_A类：被观察对象的isa指针从指向原来的A类，被KVO机制修改为指向系统新创建的子类NSKVONotifying_A类，来实现当前类属性值改变的监听。
&emsp;&emsp;子类setter方法：KVO的键值观察通知依赖于NSObject的两个方法：` willChangeValueForKey: `和` didChangeValueForKey: `，在存取数值的前后分别调用2个方法：被观察属性发生改变之前，` willChangeValueForKey: `被调用，通知系统该keyPath的属性值即将变更；当改变发生后，` didChangeValueForKey: `被调用，通知系统该keyPath的属性值已经变更；之后，` observeValueForKeyPath: ofObject: change: context:(void *)context; ` 也会被调用。

# 注意点
1. KVO的` addObserver `和` removeObserver `必须是成对的，如果重复调用remove则会导致NSRangeException类型的crash，如果忘记remove，则会在观察者释放后再次接收到KVO回调时会导致crash。
2. 观察者观察的是属性，只有遵循KVO变更属性值的方式才会执行KVO的回调方法。如果赋值没有通过setter方法或者KVC，而是直接修改属性对应的成员变量，是不会触发KVO机制，更加不会调用回调方法。所以使用KVO机制的前提是遵循KVO的属性设置方法来变更属性值。

# 使用
## 注册
&emsp;&emsp;通过` addObserver:forKeyPath:options:context: `方法注册观察者，观察者可以接收kayPath属性的变化事件。在注册观察者时，可以传入options参数，参数是一个枚举类型。如果传入` NSKeyValueObservingOptionNew `和` NSKeyValueObservingOptionOld `表示接收新值和旧值，默认为只接收新值。如果想再注册观察者后，立即接收一次回调，则可以为options参数传入` NSKeyValueObservingOptionInitial `。
&emsp;&emsp;还可以通过方法的context参数传入任意类型的对象，在接收消息回调的代码中可以接收到这个对象。

## 监听
&emsp;&emsp;观察者需要实现` observeValueForKeyPath:ofObject:change:context: `方法，当KVO时间到来时会调用这个方法，如果没有实现该方法会导致crash。change是一个字典，里面存放KVO属性相关的值，根据注册观察者时，options参数传入的枚举来返回。枚举会对应相应key来从字典中取出值，例如` NSKeyValueChangeOldKey `字段，存储改变之前的旧值。
&emsp;&emsp;如果被观察对象是集合对象，在` NSKeyValueChangeKindKey `字段中会包含` NSKeyValueChangeInsertion `、` NSKeyValueChangeRemoval `、` NSKeyValueChangeReplacement `的信息，表示集合对象的操作方式。

## 移除
&emsp;&emsp;当不需要监听或者观察者要销毁时，需要调用` removeObserver:forKeyPath: `方法，将KVO移除，否则会导致crash。

# 应用举例（Swift）
&emsp;&emsp;我们在新建项目的storyboard上，放一个UIButton和一个UILabel，新建一个继承自NSObject的model类，model中有一个num变量，当点击按钮时，使num + 1，并在controller中监听num值得改变，然后显示在UILabel上。代码如下：
```
@objcMembers class KVOModel: NSObject {
    
    dynamic var num = 0
}

class ViewController: UIViewController {

    @IBOutlet weak var numLabel: UILabel!
    
    var model = KVOModel()
    var ob: NSKeyValueObservation!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        ob = model.observe(\.num, options: [.old, .new]) { [weak self] (ob, change) in
            if let value = change.newValue {
                self?.numLabel.text = "\(value)"
            }
        }
    }

    @IBAction func addBtnAction() {
        model.num += 1
    }
}
```
&emsp;&emsp;相关说明：
- demo采用的是Swift，并且添加KVO监听的方式和OC的不一样，当然也可以用addObserver的方式。这样的好处是不需要手动remove了，但需要注意的是，观察的闭包没有被强引用，避免循环引用问题。
- 由于Swift本身屏蔽了运行时机制，只有NSObject才能支持KVO，Swift4中继承NSObject的Swift class不再默认全部bridge到OC，然而KVO又是一个纯OC的特性，所以在创建Swift class的时候需要增加**@objcMembers**关键字，另外被观察的属性也需要用**dynamic**关键字进行修饰，否则也无法观察到。
- KVO之后返回的是一个**NSKeyValueObservation**实例，需要自己控制这个实例的生命周期。
- **demo中，model的num的值直接controller中进行修改，这种做法是错误不可取的，在实际的项目中，千万不要这么使用。**

# 延伸
&emsp;&emsp;这里推荐一个第三方，Facebook开源的替代KVO的解决方案[FBKVOController](https://github.com/facebook/KVOController)，它用block解决了以前使用KVO时代码散乱的缺点，详细可以看这篇文章：[iOS开发 - 分享一个关于KVO的扩展](http://cloverkim.com/kvo_extension.html)
