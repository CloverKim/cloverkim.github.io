---
title: iOS开发 - Block用法和实现原理
categories: iOS开发
tags:
- iOS
- 基础
- 原理
top: 100
copyright: ture
---

&emsp;&emsp;《Objective-C高级编程》是一本有趣又难懂的书，全书就讲了引用计数、Block、GCD三个概念，有趣是因为讲原理、实现的部分是其他iOS专业书籍里少有的。然而每个章节不读个三五遍还是比较难理解贯通。本文针对其中的Block部分做些简单的笔记记录，讲述Block的用法和部分实现原理，详细解说从原书中寻。<!-- more -->

# Block概要
- Block：带有自动变量的匿名函数。
- 匿名函数：没有函数名的函数，一对{}包裹的内容是匿名函数的作用域。
- 自动变量：栈上声明的一个变量不是静态变量和全局变量，是不可以在这个栈内声明的匿名函数中使用的，但在Block中却可以。

&emsp;&emsp;虽然使用Block不用声明类，但是Block提供了类似Objective-C的类一样可以通过成员变量来保存作用域外变量值的方法，那些在Block的一对{}里使用到但却是在{}作用域以外声明的变量，就是Block截获的自动变量。

# Block常规概念
## Block语法
Block表达式语法：
> ^ 返回值类型(参数列表){表达式}

例如：
```
^ int (int count) {
    return count + 1;
};
```
其中，可省略部分有：
- 返回值类型，例：

```
^ (int count) {
    return count + 1;
};
```

- 参数列表为空，则可省略，例：

```
^ {
    NSLog(@"No Parameter");
};
```

即最简模式语法为：
> ^ {表达式}

## Block类型变量
声明Block类型变量语法：
> 返回值类型(^ 变量名)(参数列表) = Block表达式

例如，如下声明了一个变量名为blk的Block：
```
int (^blk)(int) = ^(int count) {
    return count + 1;
};
```

当Block类型变量作为函数的参数时，写作：
```
- (void)func:(int (^)(int))blk {
    NSLog(@"Param:%@", blk);
}
```

借助typedef，可简写：
```
typedef int (^blk_k)(int);

- (void)func:(blk_k)blk {
    NSLog(@"Param:%@", blk);
}
```

Block类型变量作返回值时，写作：
```
- (int (^)(int))funcR {
    return ^(int count) {
        return count ++;
    };
}
```

借助typedef，可简写：
```
typedef int (^blk_k)(int);

- (blk_k)funcR {
    return ^(int count) {
        return count ++;
    };
}
```

## 截获自动变量值
Block表达式可截获所使用的自动变量的值。
截获：保存自动变量的瞬间值。
因为是“瞬间值”，所以声明Block之后，即便在Block外修改自动变量的值，也不会对Block内截获的自动变量值产生影响。
例如：
```
int i = 10;
void (^blk)(void) = ^{
    NSLog(@"In block, i = %d", i);
};
i = 20;                 // Block外修改变量i，也不影响Block内的自动变量
blk();                  // i修改为20后才执行，打印: In block, i = 10
NSLog(@"i = %d", i);    // 打印：i = 20
```

## __block说明符号
&emsp;&emsp;自动变量截获的值为Block声明时刻的瞬间值，保存后就不能改写该值，如需对自动变量进行重新赋值，需要在变量声明前附加`__block`说明符，这时该变量称为`__block`变量。例如：
```
__block int i = 10;         // i为__block变量，可在block中重新赋值
void (^blk)(void) = ^{
    NSLog(@"In block, i = %d", i);
};
i = 20;
blk();                      // 打印: In block, i = 20
NSLog(@"i = %d", i);        // 打印：i = 20
```

## 自动变量值为一个对象情况
&emsp;&emsp;当自动变量为一个类的对象，且没有使用`__block`修饰时，虽然不可以在Block内对该变量进行重新赋值，但可以修改该对象的属性。
&emsp;&emsp;如果该对象是一个Mutable的对象，例如NSMutableArray，则还可以在Block内对NSMutableArray进行元素的增删，例如：
```
NSMutableArray *array = [[NSMutableArray alloc] initWithObjects:@"1", @"2",nil ];
NSLog(@"Array Count:%ld", array.count);     // 打印Array Count:2
void (^blk)(void) = ^{
    [array removeObjectAtIndex:0];          // Ok
    //array = [NSNSMutableArray new];       // 没有__block修饰，编译失败！
};
blk();
NSLog(@"Array Count:%ld", array.count);     // 打印Array Count:1
```

# Block实现原理
## 使用Clang
&emsp;&emsp;Block实际上是作为极普通的C语言源码来处理的：含有Block语法的源码首先被转换成C语言编译器能处理的源码，再作为普通的C源代码进行编译。
&emsp;&emsp;使用LLVM编译器的clang命令可将含有Block的Objective-C代码转换成C++的源代码，以探查其具体实现方式：` clang -rewrite-objc 源码文件名 `。
&emap;&emsp;祝：如果使用该命令报错：` ’UIKit/UIKit.h’ file not found `，可参考[Objective-C编译成C++代码报错](https://www.jianshu.com/p/43a09727eb2c)解决。

## Block结构
使用Block的时候，编译器对Block语言进行了怎样的转换？
```
int main() {
    int count = 10;
    void (^ blk)() = ^(){
        NSLog(@"In Block:%d", count);
    };
    blk();
}
```
如上所示的最简单的Block使用代码，经clang转换后，可得到以下几个部分（有代码删减和注释添加）：
```
static void __main_block_func_0(
    struct __main_block_impl_0 *__cself) {
    int count = __cself->count; // bound by copy
    
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_64_vf2p_jz52yz7x4xtcx55yv0r0000gn_T_main_d2f8d2_mi_0, 
    count);
}
```
这是一个函数的实现，对应Block中{}内的内容，这些内容被当做了C语言函数来处理，函数参数中的__cself相当于Objective-C中的self。
```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc; // 描述Block大小、版本等信息
  int count;
  // 构造函数函数
  __main_block_impl_0(void *fp,
          struct __main_block_desc_0 *desc,
          int _count,
          int flags=0) : count(_count) {
    impl.isa = &_NSConcreteStackBlock; // 在函数栈上声明，则为_NSConcreteStackBlock
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
` __main_block_impl_0 `即为main()函数栈上的Block结构体，其中的__block_imp结构体声明如下：
```
struct __block_impl {
  void *isa;        // 指明对象的Class
  int Flags;
  int Reserved;
  void *FuncPtr;
};
```
__block_imp结构体，即为Block的结构体，可理解为Block的类结构。
再看下main()函数翻译的内容：
```
int main() {
    int count = 10;
    void (* blk)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, count));
    
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
}
```
去除掉复杂的类型转化，可简写为：
```
int main() {
    int count = 10;
    sturct __main_block_impl_0 *blk = &__main_block_impl_0(__main_block_func_0,         // 函数指针
                                                           &__main_block_desc_0_DATA));                      // Block大小、版本等信息
    
    (*blk->FuncPtr)(blk);                         // 调用FuncPtr指向的函数，并将blk自己作为参数传入
}
```
由此看出，Block也是Objective-C中的对象。
Block有三种类（即__block_impl的isa指针指向的值，isa说明参考[Objective-C isa 指针 与 runtime 机制](https://www.jianshu.com/p/41735c66dccb)），根据Block对象创建时所处数据区不同而进行区别：
- _NSConcreteStackBlock：在栈上创建的Block对象
- _NSConcreteMallocBlock：在堆上创建的Block对象
- _NSConcreteGlobalBlock：全局数据区的Block对象

## 如何截获自动变量
&emsp;&emsp;上部分介绍了Block的结构，和作为匿名函数的调用机制，那自动变量截获是发生在什么时候呢？
&emsp;&emsp;观察上节代码中` __main_block_impl_0 `结构体（main栈上Block的结构体）的构造函数可以看到，栈上的变量count以参数的形式传入到了这个构造函数中，此处即为变量的自动截获。
&emsp;&emsp;因此可以这样理解：`__block_impl`结构体已经可以代表Block类了，但在栈上又声明了` __main_block_impl_0 `结构体，对`__block_impl`进行封装后才来表示栈上的Block类，就是为了获取Block中使用到的栈上声明的变量（栈上没在Block中使用的变量不会被捕获），变量被保存在Block的结构体实例中。
&emsp;&emsp;所以再blk()执行之前，栈上简单数据类型的count无论发生什么变化，都不会影响到Block以参数形式传入而捕获的值。但这个变量是指向对象的指针时，是可以修改这个对象的属性的，只是不能为变量重新赋值。

## Block的存储域
&emsp;&emsp;上文已提到，根据Block创建的位置不同，Block有三种类型，创建的Block对象分别会存储到栈、堆、全局数据区域。
```
void (^blk)(void) = ^{
    NSLog(@"Global Block");
};

int main() {
    blk();
    NSLog(@"%@",[blk class]);           // 打印：__NSGlobalBlock__
}
```
&emsp;&emsp;像上面代码块中的全局blk自然是存储在全局数据区，但注意在函数栈上创建的blk，如果没有截获自动变量，Block的结构实例还是会被设置在程序的全局数据区，而非栈上：
```
int main() {
    void (^blk)(void) = ^{              // 没有截获自动变量的Block
        NSLog(@"Stack Block");
    };
    blk();
    NSLog(@"%@",[blk class]);           // 打印:__NSGlobalBlock__
    
    int i = 1;
    void (^captureBlk)(void) = ^{       // 截获自动变量i的Block
        NSLog(@"Capture:%d", i);
    };
    captureBlk();
    NSLog(@"%@",[captureBlk class]);    // 打印：__NSMallocBlock__
}
```
&emsp;&emsp;可以看到截获了自动变量的Block打印的类是NSGlobalBlock，表示存储在全局数据区。但为什么捕获自动变量的Block打印的类却是设置在堆上的NSMallocBlock，而非栈上的NSStackBlock？这个问题稍后解释。

## Block复制
&emsp;&emsp;配置在栈上的Block，如果其所属的栈作用域结束，该Block就会被废弃，对于超出Block作用域仍需使用Block的情况，Block提供了将Block从栈上复制到堆上的方法来解决这种问题，即便Block栈作用域已结束，但被拷贝到堆上的Block还可以继续存在。
&emsp;&emsp;复制到堆上的Block，将_NSConcreteMallocBlock类对象写入Block结构体实例的成员变量isa：
```
impl.isa = &_NSConcreteMallocBlock;
```
&emsp;&emsp;在ARC有效时，大多数情况下编译器会进行判断，自动生成将Block从栈上复制到堆上的代码，以下几种情况栈上的Block会自动复制到堆上：
- 调用Block的copy方法
- 将Block作为函数返回值时
- 将Block赋值给__strong修改的变量时
- 向Cocoa框架含有usingBlock的方法或者GCD的API传递Block参数时

&emsp;&emsp;其他时候向方法的参数中传递Block时，需要手动调用copy方法复制Block。上一节的栈上截获了自动变量i的Block之所以在栈上创建，确实NSMallocBlock类，就是因为这个Block对象赋值给了`__strong`修饰的变量captureBlk（`__strong`是ARC下对象的默认修饰符）。
&emsp;&emsp;因为上面四条规则，在ARC下其实很少见到_NSConcreteStackBlock类的Block，大多数情况编译器都保证了Block是在堆上创建的，如下代码所示，仅最后一行代码直接使用一个不赋值给变量的Block，它的类才是NSStackBlock：
```
int count = 0;
    blk_t blk = ^(){
        NSLog(@"In Stack:%d", count);
    };
    
    NSLog(@"blk's Class:%@", [blk class]);                                      // 打印：blk's Class:__NSMallocBlock__
    NSLog(@"Global Block:%@", [^{NSLog(@"Global Block");} class]);              // 打印：Global Block:__NSGlobalBlock__
    NSLog(@"Copy Block:%@", [[^{NSLog(@"Copy Block:%d",count);} copy] class]);  // 打印：Copy Block:__NSMallocBlock__
    NSLog(@"Stack Block:%@", [^{NSLog(@"Stack Block:%d",count);} class]);       // 打印：Stack Block:__NSStackBlock__
```
&emsp;&emsp;关于ARC下和MRC下Block自动copy的区别，查看[Block 小测验](https://www.zybuluo.com/MicroCai/note/49713)里的几道题目就能区分了。
&emsp;&emsp;另外，原书存在ARC和MRC混合讲解、区分不明的情况，比如书中几个使用栈上对象导致crash的例子是MRC条件下才会发生的，但书中没做特殊说明。

## 使用__block发生了什么
&emsp;&emsp;Block捕获的自动变量添加`__block`说明符，就可在Block内读和写该变量，也可以在原来的栈上读写该变量。
&emsp;&emsp;自动变量的捕获保证了栈上的自动变量被销毁后，Block内仍可使用该变量。`__block`保证了栈上和Block内（通常在堆上）可以访问和修改“同一个变量”，`__block`是如何实现这一功能的？
&emsp;&emsp;`__block`发挥作用的原理：将栈上用`__block`修饰的自动变量封装成一个结构体，让其在堆上创建，以方便从栈上或堆上访问和修改同一份数据。
验证过程：
现在对刚才的代码段，加上`__block`说明符，并在block内外读写变量count。
```
int main() {
    __block int count = 10;
    void (^ blk)() = ^(){
        count = 20;
        NSLog(@"In Block:%d", count);       // 打印：In Block:20
    };
    count ++;
    NSLog(@"Out Block:%d", count);          // 打印：Out Block:11
    
    blk();
}
```
将上面的代码段clang，发现Block的结构体` __main_block_impl_0 `结构如下所示：
```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_count_0 *count;     // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_count_0 *_count, int flags=0) : count(_count->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```
最大的变化就是count变量不再是int类型了，count变成了一个指向` __Block_byref_count_0 `结构体的指针，` __Block_byref_count_0 `结构如下：
```
struct __Block_byref_count_0 {
  void *__isa;
__Block_byref_count_0 *__forwarding;
 int __flags;
 int __size;
 int count;
};
```
它保存了int count变量，还有一个指向` __Block_byref_count_0 `实例的指针`__forwarding`，通过下面两段代码`__forwarding`指针的用法可以知道，该指针其实指向的是对象自身：
```
// Block的执行函数
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_count_0 *count = __cself->count;    // bound by ref

        (count->__forwarding->count) = 20;          // 对应count = 20;
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_64_vf2p_jz52yz7x4xtcx55yv0r0000gn_T_main_fafeeb_mi_0, 
        (count->__forwarding->count));
    }
```
```
// main函数
int main() {
    __attribute__((__blocks__(byref))) __Block_byref_count_0 count = {(void*)0,
    (__Block_byref_count_0 *)&count, 
    0, 
    sizeof(__Block_byref_count_0), 
    10};
    
    void (* blk)() = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, 
    &__main_block_desc_0_DATA, 
    (__Block_byref_count_0 *)&count, 
    570425344));
    
    (count.__forwarding->count) ++;         // 对应count ++;
    
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_64_vf2p_jz52yz7x4xtcx55yv0r0000gn_T_main_fafeeb_mi_1, 
    (count.__forwarding->count));
    
    ((void (*)(__block_impl *))((__block_impl *)blk)->FuncPtr)((__block_impl *)blk);
}
```
为什么要通过`__forwarding`指针完成对count变量的读写修改？
为了保证无论是在栈上还是在堆上，都能通过`__forwarding`指针找到堆上创建的count这个` __main_block_func_0 `结构体，以完成对count->count（第一个count是` __main_block_func_0 `对象，第二个count是int类型变量）的访问和修改。示意图如下：
![](http://pz1livcqe.bkt.clouddn.com/006tNc79gy1g03s2zred9j30xc0o1408.jpg)

## Block的循环引用
&emsp;&emsp;Block的循环引用原理和解决方法大家都比较熟悉，此处将结合上文的介绍，介绍一种不常用的解决Block循环引用的方法和一种借助Block参数该问题的方法。
&emsp;&emsp;Block循环引用的原因：一个对象A有Block类型的属性，从而持有这个Block，如果Block的代码块中使用到这个对象A，或者仅仅是用到A对象的属性，会使Block也持有A对象，导致两者互相持有，不能在作用域结束后正常释放。
&emsp;&emsp;解决原理：对象A照常持有Block，但Block不能强引用持有对象A以打破循环。
### 第一种解决方法
&emsp;&emsp;对Block内部使用的对象A使用`__weak`进行修饰，Block对对象A弱引用打破循环。有以下三种常用形式：
- 使用__weak ClassName

    ```
    __block XXViewController *weakSelf = self;
    self.blk = ^{
        NSLog(@"In Block : %@",weakSelf);
    };
    ```
    
- 使用__weak typeof(self)

    ```
    __weak typeof(self) weakSelf = self;
    self.blk = ^{
        NSLog(@"In Block : %@",weakSelf);
    };
    ```
    
- Reactive Cocoa中的@weakify和@strongify

    ```
    @weakify(self);
    self.blk = ^{
        @strongify(self);
        NSLog(@"In Block : %@",self);
    };
    ```
    其原理参考（[@weakify, @strongify](https://www.jianshu.com/p/3d6c4416db5e)），已经简便实现参考（[@weak - @strong 宏的实现](https://blog.csdn.net/u014773226/article/details/54617716)）。
    
### 第二种解决方法
&emsp;&emsp;对Block内要使用的对象A使用`__block`进行修饰，并在代码块内，使用完`__block`变量后将其设为nil，并且该block必须至少执行一次。
```
__block XXController *blkSelf = self;
self.blk = ^{
    NSLog(@"In Block : %@",blkSelf);
};
```
注意上述代码仍存在内存泄漏，因为：
- XXController对象持有Block对象blk
- blk对象持有`__block`变量blkSelf
- `__block`变量blkSelf持有XXController对象

 ```
__block XXController *blkSelf = self;
    self.blk = ^{
        NSLog(@"In Block : %@",blkSelf);
        blkSelf = nil;      // 不能省略
    };
    
    self.blk();             // 该block必须执行一次，否则还是内存泄露
```
在block代码块内，使用完`__block`变量后将其设为nil，并且该block必须至少执行一次后，不存在内存泄漏，因为此时：
 - XXController对象持有Block对象blk
 - blk对象持有`__block`变量blkSelf（类型为编译器创建的结构体）
 - `__block`变量blkSelf在执行blk()之后被设置为nil（`__block`变量结构体的`__forwarding`指针指向了nil），不再持有XXController对象，打破循环。
 
&emsp;&emsp;第二种使用`__block`打破循环的方法，优点是：
- 可通过`__bloc`k变量动态控制持有XXController对象的时间，运行时决定是否将nil或其他变量赋值给`__block`变量
- 不能使用`__wea`k的系统中，使用`__unsafe)unretaines`来代替`__weak`打破循环可能由野指针问题，使用`__block`则可避免该问题。

&emsp;&emsp;其缺点也明显：
- 必须手动保证`__block`变量最后设置为nil
- block必须执行一次，否则`__block`不为nil循环引用仍存在

&emsp;&emsp;因此，还是避免使用第二种不常用方式，直接使用`__weak`打破Block循环引用。

### 第三种解决方法
&emsp;&emsp;将在Block内要使用到的对象（一般为self），以Block参数的形式传入，Block就不会捕获该对象，而将其作为参数使用，其生命周期系统的栈自动管理，不造成内存泄漏。
即使用原来`__weak`的写法：
```
__weak typeof(self) weakSelf = self;
    self.blk = ^{
        __strong typeof(self) strongSelf = weakSelf;
        NSLog(@"Use Property:%@", strongSelf.name);
        //……
    };
    self.blk();
```
改为Block传参写法后：
```
self.blk = ^(UIViewController *vc) {
        NSLog(@"Use Property:%@", vc.name);
    };
    self.blk(self);
```
优点:
- 简化了两行代码，更优雅
- 更明确的API设计：告诉API使用者，该方法多的Block直接使用传进来的参数对象，不会造成循环引用，不用调用者再使用weak避免循环引用。

该种用法的详细思路，和clang后的数据结构，可参考（[Heap-Stack Dance](https://github.com/ChenYilong/iOSBlog/blob/master/Tips/Heap-Stack%20Dance/Heap-Stack%20Dance.md)）

# 转载
&emsp;&emsp;本文转自[简书 - kamous - iOS Block用法和实现原理](https://www.jianshu.com/p/d28a5633b963)