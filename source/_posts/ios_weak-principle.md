---
title: iOS开发 - 底层解析weak的实现原理
categories: iOS开发
tags:
- iOS
- 原理
top: 100
copyright: ture
---

# 前言
&emsp;&emsp;很少有人知道weak表其实是一个hash（哈希）表，key是所指对象的地址，value是weak指针的地址数组。大多数人只知道weak是弱引用，所引用对象的计数器不会加一，并在引用对象被释放的时候自动被置为nil。通常用于解决循环引用问题。但现在单知道这些已经不足以应对面试了，很多公司都会问weak的原理。weak的原理是什么？下面就分析一下weak的原理。
<!-- more -->
# weak 实现原理的概括
&emsp;&emsp;runtime维护了一个weak表，用于存储指向某个对象的所有weak指针。weak表其实是一个hash（哈希）表，key是所指对象的地址，value是weak指针的地址（这个地址的值是所指对象指针的地址）数组。

# weak 的实现原理详细步骤
## (一)、初始化
&emsp;&emsp;初始化时，runtime会调用objc_initWeak函数，初始化一个新的weak指针指向对象的地址。
示例代码：
```
NSObject *obj = [[NSObject alloc] init];
id __weak obj1 = obj;
```
&emsp;&emsp;在我们初始化一个weak变量时，runtime会调用NSObject.mm中的objc_initWeak函数，这个函数在Clang中的声明如下：
```
id objc_initWeak(id *object, id value);
```
而对于objc_initWeak()方法的实现
```
id objc_initWeak(id *location, id newObj) {
// 查看对象实例是否有效
// 无效对象直接导致指针释放
    if (!newObj) {
        *location = nil;
        return nil;
    }
    // 这里传递了三个 bool 数值
    // 使用 template 进行常量参数传递是为了优化性能
    return storeWeakfalse/*old*/, true/*new*/, true/*crash*/>
    (location, (objc_object*)newObj);
}
```
&emsp;&emsp;可以看出，这个函数仅仅是一个深层函数的调用入口，而一般的入口函数中，都会做一些简单的判断（例如objc_msgSend中的缓存判断），这里判断了其指针指向的类对象是否有效，无效直接释放，不再往深层调用函数。否则，object将被注册为一个指向value的__weak对象。
&emsp;&emsp;注意：objc_initWeak函数有一个前提条件：就是object必须是一个没有被注册为__weak对象的有效指针。而value则可以为空，或者指向一个有效的对象。

## (二)、添加引用
&emsp;&emsp;添加引用时：objc_initWeak函数会调用objc_storeWeak()函数，objc_storeWeak()的作用是更新指针指向，创建对应的弱引用表。
objc_storeWeak的函数声明和具体实现如下：
```
id objc_storeWeak(id *location, id value);
```
```
// HaveOld:  true - 变量有值
//          false - 需要被及时清理，当前值可能为 nil
// HaveNew:  true - 需要被分配的新值，当前值可能为 nil
//          false - 不需要分配新值
// CrashIfDeallocating: true - 说明 newObj 已经释放或者 newObj 不支持弱引用，该过程需要暂停
//          false - 用 nil 替代存储
template bool HaveOld, bool HaveNew, bool CrashIfDeallocating>
static id storeWeak(id *location, objc_object *newObj) {
    // 该过程用来更新弱引用指针的指向
    // 初始化 previouslyInitializedClass 指针
    Class previouslyInitializedClass = nil;
    id oldObj;
    // 声明两个 SideTable
    // ① 新旧散列创建
    SideTable *oldTable;
    SideTable *newTable;
    // 获得新值和旧值的锁存位置（用地址作为唯一标示）
    // 通过地址来建立索引标志，防止桶重复
    // 下面指向的操作会改变旧值
retry:
    if (HaveOld) {
        // 更改指针，获得以 oldObj 为索引所存储的值地址
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (HaveNew) {
        // 更改新值指针，获得以 newObj 为索引所存储的值地址
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }
    // 加锁操作，防止多线程中竞争冲突
    SideTable::lockTwoHaveOld, HaveNew>(oldTable, newTable);
    // 避免线程冲突重处理
    // location 应该与 oldObj 保持一致，如果不同，说明当前的 location 已经处理过 oldObj 可是又被其他线程所修改
    if (HaveOld  &&  *location != oldObj) {
        SideTable::unlockTwoHaveOld, HaveNew>(oldTable, newTable);
        goto retry;
    }
    // 防止弱引用间死锁
    // 并且通过 +initialize 初始化构造器保证所有弱引用的 isa 非空指向
    if (HaveNew  &&  newObj) {
        // 获得新对象的 isa 指针
        Class cls = newObj->getIsa();
        // 判断 isa 非空且已经初始化
        if (cls != previouslyInitializedClass  &&
            !((objc_class *)cls)->isInitialized()) {
            // 解锁
            SideTable::unlockTwoHaveOld, HaveNew>(oldTable, newTable);
            // 对其 isa 指针进行初始化
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));
            // 如果该类已经完成执行 +initialize 方法是最理想情况
            // 如果该类 +initialize 在线程中
            // 例如 +initialize 正在调用 storeWeak 方法
            // 需要手动对其增加保护策略，并设置 previouslyInitializedClass 指针进行标记
            previouslyInitializedClass = cls;
            // 重新尝试
            goto retry;
        }
    }
    // ② 清除旧值
    if (HaveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }
    // ③ 分配新值
    if (HaveNew) {
        newObj = (objc_object *)weak_register_no_lock(&newTable->weak_table,
                                                      (id)newObj, location,
                                                      CrashIfDeallocating);
        // 如果弱引用被释放 weak_register_no_lock 方法返回 nil
        // 在引用计数表中设置若引用标记位
        if (newObj  &&  !newObj->isTaggedPointer()) {
            // 弱引用位初始化操作
            // 引用计数那张散列表的weak引用对象的引用计数中标识为weak引用
            newObj->setWeaklyReferenced_nolock();
        }
        // 之前不要设置 location 对象，这里需要更改指针指向
        *location = (id)newObj;
    } else {
        // 没有新值，则无需更改
    }
    SideTable::unlockTwoHaveOld, HaveNew>(oldTable, newTable);
    return (id)newObj;
}
```
撇开源码中各种锁操作，来看看这段代码都做了些什么：
1. SideTable
&emsp;&emsp;SideTable这个结构体，引用计数和弱引用依赖表，主要用于管理对象的引用计数和weak表，在NSObject.m中声明其数据结构：
```
struct SideTable {
// 保证原子操作的自旋锁
    spinlock_t slock;
    // 引用计数的 hash 表
    RefcountMap refcnts;
    // weak 引用全局 hash 表
    weak_table_t weak_table;
}
```
&emsp;&emsp;对于slock和refcnts，第一个是为了防止竞争选择的自旋锁，第二个是协助对象的isa纸巾的extra_rc共同引用计数的变量。

2. weak表
&emsp;&emsp;weak表是一个弱引用表，实现为一个weak_table_t结构体，存储了某个对象相关的所有的弱引用信息，其定义如下（objc_weak.h）：
```
struct weak_table_t {
    // 保存了所有指向指定对象的 weak 指针
    weak_entry_t *weak_entries;
    // 存储空间
    size_t    num_entries;
    // 参与判断引用计数辅助量
    uintptr_t mask;
    // hash key 最大偏移值
    uintptr_t max_hash_displacement;
};
```
&emsp;&emsp;这是一个全局弱引用hash表，使用不定类型对象的地址作为key，用weak_entry_t类型结构体对象作为value。其中的weak_entries成员，从字面意思看，即为弱引用表入口。
&emsp;&emsp;其中weak_entry_t是存储在弱引用表中的一个内部结构体，它负责维护和存储指向一个对象的所有弱引用hash表。其定义如下：
```
typedef objc_object ** weak_referrer_t;
struct weak_entry_t {
    DisguisedPtr<objc_object> referent;
    union {
        struct {
            weak_referrer_t *referrers;
            uintptr_t        out_of_line : 1;
            uintptr_t        num_refs : PTR_MINUS_1;
            uintptr_t        mask;
            uintptr_t        max_hash_displacement;
        };
        struct {
            // out_of_line=0 is LSB of one of these (don't care which)
            weak_referrer_t  inline_referrers[WEAK_INLINE_COUNT];
        };
    }
}
```
&emsp;&emsp;在weak_entry_t的结构体中，DisguisedPtrobjc_object>是对泛型对象的指针做了一个封装，通过这个泛型类来解决内存泄漏的问题。从注释中，写out_of_line成员为最低有效位，当其为0的时候，weak_referrer_t成员将扩展为多行静态hash table。其实其中的weak_referrer_t是二维objc_object的别名，通过一个二维指针地址偏移，用下标作为hash的key，做成了一个弱引用散列。

3. 旧对象解除注册操作weak_unregister_no_lock
&emsp;&emsp;该方法主要作用是将旧对象在weak_table中解除weak指针的对应绑定。根据函数名，称之为解除注册操作。从源码中，可以知道其功能就是从weak_table中解除weak指针的绑定。而其中的遍历查询，就是针对于weak_entry中的多张弱引用散列表。

4. 新对象添加注册操作weak_register_no_lock
&emsp;&emsp;这一步与上一步相反，通过weak_register_no_lock函数把新的对象进行注册操作，完成与对应的弱引用表进行绑定操作。

5. 初始化弱引用对象流程一览
&emsp;&emsp;弱引用初始化，从上文的分析中可以看出，主要的操作部分就在弱引用表的取键、查询散列、创建弱引用表等操作，可以总结出如下的流程图：
![](https://ws1.sinaimg.cn/large/749c46aagy1fxljq6knw6j20sg0sgwh5.jpg) 

## (三)、释放
&emsp;&emsp;释放时，调用clearDeallocating函数。clearDeallocating函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。
&emsp;&emsp;当weak引用指向的对象被释放时，又是如何去处理weak指针的呢？当释放对象时，其基本流程如下：
1. 调用objc_release
2. 因为对象的引用计数为0，所以执行dealloc函数
3. 在dealloc中，调用了_objc_rootDealloc函数
4. 在_objc_rootDealloc中，调用了object_disponse函数
5. 调用objc_destructInstance
6. 最后调用objc_clear_deallocating

&emsp;&emsp;重点看对象被释放时调用的objc_clear_deallocation函数，该函数的实现如下：
```
void  objc_clear_deallocating(id obj) 
{
    assert(obj);
    assert(!UseGC);
    if (obj->isTaggedPointer()) return;
   obj->clearDeallocating();
}
```
&emsp;&emsp;也就是调用了clearDeallocating函数，继续追踪可以发现，它最终是使用了迭代器来取weak表的value，调用weak_clear_no_lock，然后查找对应的value，将该weak指针置空，weak_clear_no_lock函数的实现如下：
```
/**
 * Called by dealloc; nils out all weak pointers that point to the
 * provided object so that they can no longer be used.
 *
 * @param weak_table
 * @param referent The object being deallocated.
 */
void weak_clear_no_lock(weak_table_t *weak_table, id referent_id)
{
    objc_object *referent = (objc_object *)referent_id;
    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }
    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    
    if (entry->out_of_line) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    }
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n",
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    weak_entry_remove(weak_table, entry);
}
```
&emsp;&emsp;objc_clear_deallocating函数的动作如下：
1. 从weak表中获取废弃对象的地址为键值得记录
2. 将包含在 记录中的所有附有weak修饰符变量的地址，赋值为nil
3. 将weak表中删除该记录
4. 从引用计数表中删除废弃对象的地址为键值得记录

&emsp;&emsp;看了objc_weak.mm的源码就明白了：其实weak表是一个hash（哈希）表，key是指向对象的地址，value是weak指针的地址数组。

# 转载于
> 标题：iOS 底层解析weak的实现原理（包含weak对象的初始化，引用，释放的分析）
> 作者：逍遥晨旭
> 链接：https://www.jianshu.com/p/13c4fb1cedea
> 來源：简书