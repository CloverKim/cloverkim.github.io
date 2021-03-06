---
title: iOS开发 - KVC原理分析
categories: iOS开发
tags:
- iOS
- 基础
top: 100
copyright: ture
---

# 定义
&emsp;&emsp;KVC（全称key-value coding）键值编码。在iOS开发中，允许开发者通过key直接访问对象的属性，或者给对象的属性进行赋值，而不需要调用明确的存取方法。这样就可以在运行时动态的访问和修改对象的属性，而不是在编译时确定。
&emsp;&emsp;KVC的定义是通过对NSObject的扩展来实现的，定义在NSKeyValueCoding.h文件中，是一个非正式协议。
<!-- more -->
# KVC相关方法
&emsp;&emsp;在NSKeyValueCoding中，KVC最为重要的方法如下：
```
// 通过key来取值
- (id)valueForKey:(NSString *)key;

// 通过keyPath来取值
- (id)valueForKeyPath:(NSString *)keyPath;

// 通过key来设值
- (void)setValue:(id)value forKey:(NSString *)key;

// 通过keyPath来设值
- (void)setValue:(id)value forKeyPath:(NSString *)keyPath;
```

&emsp;&emsp;NSKeyValueCoding中还有其他的相关方法，例如：
```
// KVC提供属性值确认的API，它可以用来检查set的值是否正确，为不正确的值做一个替换值或者拒绝设值新值并返回错误原因
- (BOOL)validateValue:(inout id  _Nullable *)ioValue forKey:(NSString *)inKey error:(out NSError * _Nullable *)outError;

// 如果key不存在，且没有KVC无法搜索到任何和key有关的字段或者属性，则会调用这个方法，默认是抛出异常
- (void)setValue:(id)value forUndefinedKey:(NSString *)key;

// 和上一个方法一样，上一个方法为设值，该方法为取值
- (id)valueForUndefinedKey:(NSString *)key;

// 如果在setValue方法时给value传nil，则会调用该方法
- (void)setNilValueForKey:(NSString *)key;

// 输入一组key，返回该组key对应的value，再转成字典返回，用于将model转字典
- (NSDictionary<NSString *,id> *)dictionaryWithValuesForKeys:(NSArray<NSString *> *)keys;
```

# 寻找key的策略
## setValue:forKey:方法赋值的原理
&emsp;&emsp;设值会调用`setValue:forKey:`方法，其大致步骤如下流程图所示：
![](http://pic.cloverkim.com/setValue-forKey-流程图.jpg 'setValue:forKey:流程图')

1. 查找`set<Key>:`或`_set<Key>:`命名的setter，按照这个顺序，如果找到，则调用这个方法并将值传进去。
2. 如果没有发现一个简单的setter，但是`accessInstanceVariablesDirectly`类属性返回YES，则查找一个命名规则为_key、_isKey、key、isKey的实例变量。按照这个顺序，如果查找到则将value赋值给实例变量。
3. 如果没有找到setter或实例变量，则调用`setValue:forUndefinedKey:`方法，并默认抛出一个异常。

## valueForKey:方法取值的原理
&emsp;&emsp;当调用`valueForKey:`方法时，KVC对key的搜索顺序有点不同于`setValue:forKey:`方法，大致步骤如下：
1. 首先按`get<Key>`、`<key>`、`is<Key>`的顺序查找getter方法，找到直接调用。
    - 若方法的返回结果类型是一个对象指针，则直接返回结果。
    - 若类型为能够转化为NSNumber的基本数据类型，转换为NSNumber后返回；否则转换为NSValue返回。
2. 若上面的getter没有找到，则查找`countOf<Key>`、`objectIn<Key>AtIndex:`、`<Key>AtIndexes`格式的方法。
如果`countOf<Key>`和另外两个方法中的一个找到，那么就会返回一个可以响应NSArray所有方法的集合代理。发送给这个代理集合的NSArray消息方法，就会以`countOf<Key>`、`objectIn<Key>AtIndex:`、`<Key>AtIndexes`这几个方法组合的形式调用。如果receiver的类实现了`get<Key>:range:`方法，该方法也会用于性能优化。

3. 还没查到，那么查找`countOf<Key>`、`enumeratorOf<Key>`、`memberOf<Key>:`格式的方法。如果这3个方法都找到，那么久返回一个可以相应NSSet所有方法的集合代理。发送给这个代理集合的NSSet消息方法，就会以`countOf<Key>`、`enumeratorOf<Key>`、`memberOf<Key>:`组合的形式调用。

4. 还是没查到，那么如果类方法`accessInstanceVariablesDirectly`返回YES，那么按`_<key>`、`_is<Key>`、`<key>`、`is<Key>`的顺序直接搜索实例变量。如果搜索到了，则返回receiver相应实例变量的值。

5. 再没有查到，调用`valueForUndefinedKey:`方法，抛出异常。

# 使用keyPath
&emsp;&emsp;在实际开发过程中，一个类的成员变量有可能是自定义类或者其他的复杂数据类型，我们可以先用KVC获取该属性，然后再用KVC来获取这个自定义类的属性。但这样比较繁琐，因此KVC提供了一个解决方案，keyPath。
```
- (nullable id)valueForKeyPath:(NSString *)keyPath;                  //通过KeyPath来取值
- (void)setValue:(nullable id)value forKeyPath:(NSString *)keyPath;  //通过KeyPath来设值
```

# 处理异常
&emsp;&emsp;使用KVC过程中最常见的异常就是不小心使用了错误的key，或者在设值时不小心传了nil的值，KVC有特定的方法处理这些异常。
- KVC处理nil异常，如果在设值过程中，不小心传了nil值，KVC会调用方法`setNilValueForKey:`，这个默认方法是抛出`NSInvalidArgumentException`异常，所以一般而言最好重写这个方法，对异常进行处理。
- KVC处理UndefinedKey异常，如果在设值取值传的key不存在时，程序就会crash，设值会调用到`setValue:forUndefinedKey:`方法，而取值会调用`valueForUndefinedKey:`方法，这两个方法默认都是抛出`NSUndefinedKeyException`异常，因此如果要避免程序crash，可以重写这两个方法。

# 集合类运算
## 集合运算符格式
&emsp;&emsp;KVC提供的`valueForKeyPath:`方法非常强大，可以通过该方法对集合对象进行“深入”操作，在其keyPath中嵌套集合运算符，例如求一个数组中对象某个属性的count。集合运算符的格式如下：
```
keyPathToCollection.@collentionOperator.keyPathToproperty
```
- keyPathToCollection：Left key path，要操作的集合对象，若调用`valueForKeyPath:`方法的对象本来就是集合对象，则可以为空。
- collectionOperator：Collection operator，集合操作符，一般以@开头。
- keyPathToproperty：Right key path，要运算的属性。

## 集合运算符的分类
&emsp;&emsp;集合运算符主要分为以下三类：
- 集合操作符：处理集合包含的对象，并根据操作符的不同返回不同的类型，返回值以NSNumber为主。
- 数组操作符：根据操作符的条件，将符合条件的对象包含在数组中返回。
- 嵌套操作符：处理集合对象中嵌套其他集合对象的情况，返回结果也是一个集合对象。

### 集合操作符
&emsp;&emsp;为了演示集合操作符，我们新建一个项目，定义一个Book类，有bookName和bookPrice属性，然后在main函数中，新建一个Book数组，再对数组进行集合操作。详细操作如下：
- `@avg`用来计算集合中`right keyPath`指定的属性的平均值。
```
NSNumber *avgNum = [bookrack valueForKeyPath:@"@avg.bookPrice"];
NSLog(@"avg: %f", [avgNum floatValue]);
```

- `@count`用来计算集合中对象的数量。注意：@count操作符不需要写rightKeyPath，如果写了也会被忽略。
```
NSNumber *count = [bookrack valueForKeyPath:@"@count"];
NSLog(@"count: %f", [count floatValue]);
```

- `@sum`用来计算集合中`right keyPath`指定的属性的总和。
```
NSNumber *sum = [bookrack valueForKeyPath:@"@sum.bookPrice"];
NSLog(@"sum: %f", [sum floatValue]);
```

- `@max`用来查找集合中`right keyPath`指定属性的最大值。
```
NSNumber *max = [bookrack valueForKeyPath:@"@max.bookPrice"];
NSLog(@"max: %f", [max floatValue]);
```

- `@min`用来查找集合中`right keyPath`指定属性的最小值。
```
NSNumber *min = [bookrack valueForKeyPath:@"@min.bookPrice"];
NSLog(@"min: %f", [min floatValue]);
```

### 数组操作符
- `@unionOfObjects`将集合中的所有对象的同一个属性放在数组中返回。
```
NSArray *priceArray = [bookrack valueForKeyPath:@"@unionOfObjects.bookPrice"];
NSLog(@"unionOfObjects: %@", priceArray);
```

- `@distinctUnionOfObjects`将集合中对象的属性进行去重后并返回。
```
NSArray *nameArray = [bookrack valueForKeyPath:@"@distinctUnionOfObjects.bookName"];
NSLog(@"distinctUnionOfObjects: %@", nameArray);
```
&emsp;&emsp;需要注意：以上两个方法，如果操作的属性为nil，则在添加到数组中时会导致crash。

### 嵌套操作符
&emsp;&emsp;由于嵌套操作符是需要对嵌套的集合对象进行操作，所以新建了一个racks数组，其中包含了两个Book类型对象的数组。
- `@unionOfArrays`是用来操作集合内部的集合对象，将所有`right keyPath`对应的对象放在一个数组中返回。
```
NSArray *unionArray = [racks valueForKeyPath:@"@unionOfArrays.bookName"];
NSLog(@"unionOfArrays: %@", unionArray);
```

- `@distinctUnionOfArrays`是用来操作集合内部的集合对象，将所有`right keyPath`对应的对象放在一个数组中，并进行去重后返回。
```
NSArray *distinctArray = [racks valueForKeyPath:@"@distinctUnionOfArrays.bookPrice"];
NSLog(@"distinctUnionOfArrays: %@", distinctArray);
```

# KVC 安全性检查
&emsp;&emsp;在使用KVC时，由于传入的key或者keyPath是一个字符串，因此很容易写错或者属性本身修改后忘记修改对应的字符串，导致crash。
&emsp;&emsp;解决的方案为，利用反射机制，通过`@selector()`获取到方法的SEL，然后通过`NSStringFromSelector()`将SEL反射为字符串。这样在`@selector()`中传入方法名的过程中，编译器会有合法性检查，如果方法不存在或者未实现时，会报对应的警告。
```
[self valueForKey:NSStringFromSelector(@selector(object))];
```

# 参考
- [Apple官方文档](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/KeyValueCoding/BasicPrinciples.html#//apple_ref/doc/uid/20002170-BAJEAIEE)
- [简书 - 拧发条鸟xds - iOS 关于KVC的一些总结](https://www.jianshu.com/p/f6c41ffb88df)
- [个人blog - 李峰峰博客 - iOS KVC详解](https://imlifengfeng.github.io/article/493/)
- [简书 - 刘小壮 - KVC原理剖析](https://www.jianshu.com/p/1d39bc610a5b)