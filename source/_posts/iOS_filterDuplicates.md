---
title: iOS开发 - Swift数组去重
categories: iOS开发
tags:
- iOS
- Swift
top: 100
copyright: ture
---

&emsp;&emsp;在开发过程中，也许会遇到需要对数组进行去重的相关处理。如果数组内只含有基础类型的数据时，你可以写两个for循环遍历，用下标取值做对比；当然也可以用集合Set（Swift），比较方便快捷，可以参照这篇文章：[Swift 中超快捷去重方法(附集合Set的一点干货)](http://www.jianshu.com/p/dcae25a96f4c)。
但如果需要对model数组进行去重，该怎么做呢？请往下看~
<!-- more -->
# Swift代码实现：
```
//: Playground - noun: a place where people can play

import UIKit

extension Array {
    
    // 去重
    func filterDuplicates<E: Equatable>(_ filter: (Element) -> E) -> [Element] {
        var result = [Element]()
        for value in self {
            let key = filter(value)
            if !result.map({filter($0)}).contains(key) {
                result.append(value)
            }
        }
        return result
    }
}

class DemoModel: CustomStringConvertible {
    
    let name: String

    init(_ name: String) {
        self.name = name
    }
    
    var description: String {
        return name
    }
}

let arrays = ["1", "2", "2", "3", "4", "4"]
let filterArrays = arrays.filterDuplicates({$0})
print(filterArrays)

let modelArrays = [DemoModel("1"), DemoModel("1"), DemoModel("2"), DemoModel("3")]
let filterModels = modelArrays.filterDuplicates({$0.name})
print(filterModels)
```
# 相关说明：
- 上面的代码是一个playground，可以用Xcode创建一个playground，将以上代码粘贴到playground即可。playground用来写一些测试函数还是挺有用的。
- filterDuplicates这个方法，这里直接写在Array的扩展里面，这样一个数组就可以随意调用这个方法了，相当的方便。
- 测试代码中，第一个是数组装的是String类型，可以直接用其值作为判断条件是否有重复值，是否需要去重。第二个是我们自定义的demoModel，有个name属性，那我们就可以用这个属性作为是否需要去重的判断，当然肯定不能直接根据一个类来判断。如果是开发中，model类肯定有很多的属性，如果要判断去重的话，需要一个不会有相同值的属性，比如iD什么的，来进行判断。
- [CustomStringConvertible](https://developer.apple.com/reference/swift/customstringconvertible)。是一个协议，实现这个协议，就可以用description来自定义print输出的内容。