---
title: SDWebImage相关面试题
categories: iOS开发
tags:
- iOS
- 面试题
top: 100
copyright: ture
---

# 概述
&emsp;&emsp;在之前的三篇文章中（[SDWebImage源码解析（一）](http://cloverkim.com/SDWebImage-source-code-analysis-1.html)、[SDWebImage源码解析（二）](http://cloverkim.com/SDWebImage-source-code-analysis-2.html)、[SDWebImage源码解析（三）](http://cloverkim.com/SDWebImage-source-code-analysis-3.html)），对[SDWebImage](https://github.com/SDWebImage/SDWebImage)的源码进行了详细的解析，阅读优秀的源码是一种很好的学习方式，对自己的技术成长有着一定的帮助。本文中，是对网上一些关于SDWebImage面试题的相关列举和分析解答。<!-- more -->

# SDWebImage支持GIF图吗？
## FLAnimatedImage
&emsp;&emsp;在SDWebImage 4.0版本之后，可以用[FLAnimatedImage](https://github.com/Flipboard/FLAnimatedImage)来加载GIF图，官方文档说明如下：
> Starting with the 4.0 version, we rely on [FLAnimatedImage](https://github.com/Flipboard/FLAnimatedImage) to take care of our animated images.
> If you use cocoapods, add `pod 'SDWebImage/GIF'` to your podfile.
> To use it, simply make sure you use FLAnimatedImageView instead of UIImageView.

> Note: there is a backwards compatible feature, so if you are still trying to load a GIF into a UIImageView, it will only show the 1st frame as a static image by default. However, you can enable the full GIF support by using the built-in GIF coder. See [GIF coder](https://github.com/SDWebImage/SDWebImage/wiki/Advanced-Usage#gif-coder)


> Important: FLAnimatedImage only works on the iOS platform. For macOS, use NSImageView with animates set to YES to show the entire animated images and NO to only show the 1st frame. For all the other platforms (tvOS, watchOS) we will fallback to the backwards compatibility feature described above

FLAnimatedImage使用流程大致可以概括为：
- CocoaPods单独倒入库`pod 'SDWebImage/GIF'`,并引入`import FLAnimatedImage`
- 使用`initWithAnimatedGIFData:`方法新建FLAnimatedImage类
- 然后再新建FLAnimatedImageView类，将刚刚新建的FLAnimatedImage赋值给FLAnimatedImageView的animatedImage变量即可。

简易demo代码如下：
```
let gifPath = Bundle.main.path(forResource: "gif_test", ofType: "gif")
if let imageData = NSData(contentsOfFile: gifPath ?? "") {
    let animatedImageView = FLAnimatedImageView(frame: animatedBgView.bounds)
    animatedImageView.animatedImage = FLAnimatedImage(animatedGIFData: imageData as Data)
    animatedBgView.addSubview(animatedImageView)
}
```

## SDWebImage 4.4.2版本的`sd_animatedGIFWithData`方法
&emsp;&emsp;在SDWebImage的4.4.2版本中，已经支持加载GIF图，使用的是`UIImage+GIF`分类中的`sd_animatedGIFWithData`方法，具体使用方法如下demo所示：
```
let gifPath = Bundle.main.path(forResource: "gif_test", ofType: "gif")
if let imageData = NSData(contentsOfFile: gifPath ?? "") {
    gifImageView.image = UIImage.sd_animatedGIF(with: imageData as Data)
}
```
运行结果如下图所示：
![](http://pic.cloverkim.com/006tKfTcgy1g0e679vq7dg30aw0lkkjm.gif)

# SDWebImage如何区分图片格式？
常见的三种图片格式：
- PNG：压缩比没有JPG高，但是无损压缩，解压缩性能高，苹果推荐的图片格式
- JPG：压缩比最高的一种图片格式，有损压缩。
- GIF：序列帧动图。特点：只支持256种颜色。

在`NSData+ImageContentType`分类中，对图片格式枚举的定义，以及根据图片data获取图片类型的定义实现如下：
```
typedef NS_ENUM(NSInteger, SDImageFormat) {
    SDImageFormatUndefined = -1,        // 未知类型
    SDImageFormatJPEG = 0,              // JPG
    SDImageFormatPNG,                   // PNG
    SDImageFormatGIF,                   // GIF
    SDImageFormatTIFF,                  // TIFF
    SDImageFormatWebP,                  // WEBP
    SDImageFormatHEIC                   // HEIC
};

/// 根据图片NSData获取图片的类型
+ (SDImageFormat)sd_imageFormatForImageData:(nullable NSData *)data {
    if (!data) {
        return SDImageFormatUndefined;
    }
    
    // File signatures table: http://www.garykessler.net/library/file_sigs.html
    uint8_t c;
    // 获取图片data数据的第一个字节数据
    [data getBytes:&c length:1];
    // 根据字母的ASCII码比较，返回对应的图片类型枚举
    switch (c) {
        case 0xFF:
            return SDImageFormatJPEG;
        case 0x89:
            return SDImageFormatPNG;
        case 0x47:
            return SDImageFormatGIF;
        case 0x49:
        case 0x4D:
            return SDImageFormatTIFF;
        case 0x52: {
            if (data.length >= 12) {
                // RIFF....WEBP
                NSString *testString = [[NSString alloc] initWithData:[data subdataWithRange:NSMakeRange(0, 12)] encoding:NSASCIIStringEncoding];
                if ([testString hasPrefix:@"RIFF"] && [testString hasSuffix:@"WEBP"]) {
                    return SDImageFormatWebP;
                }
            }
            break;
        }
        case 0x00: {
            if (data.length >= 12) {
                // ....ftypheic ....ftypheix ....ftyphevc ....ftyphevx
                NSString *testString = [[NSString alloc] initWithData:[data subdataWithRange:NSMakeRange(4, 8)] encoding:NSASCIIStringEncoding];
                if ([testString isEqualToString:@"ftypheic"]
                    || [testString isEqualToString:@"ftypheix"]
                    || [testString isEqualToString:@"ftyphevc"]
                    || [testString isEqualToString:@"ftyphevx"]) {
                    return SDImageFormatHEIC;
                }
            }
            break;
        }
    }
    return SDImageFormatUndefined;
}
```

# SDWebImage缓存图片的名称如何避免重名
&emsp;&emsp;如果单纯使用文件名保存，重名的几率非常高。因此，使用MD5的散列函数，对完整的图片url进行MD5，结果是一个32个字符长度的字符串。

# SDWebImage如何保证UI操作放在主线程中执行？
&emsp;&emsp;在SDWebImage的`SDWebImageCompat.h`中，有如下的宏定义，用来保证主线程操作，可为什么要这么写？
```
#ifndef dispatch_queue_async_safe
#define dispatch_queue_async_safe(queue, block)\
    if (dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL) == dispatch_queue_get_label(queue)) {\
        block();\
    } else {\
        dispatch_async(queue, block);\
    }
#endif

#ifndef dispatch_main_async_safe
#define dispatch_main_async_safe(block) dispatch_queue_async_safe(dispatch_get_main_queue(), block)
#endif
```
之前比较常见的写法如下：
```
#define dispatch_main_async_safe(block)\
    if ([NSThread isMainThread]) {\
        block();\
    } else {\
        dispatch_async(dispatch_get_main_queue(), block);\
    }
```
&emsp;&emsp;对比两个宏定义可以发现前者两个地方改变了，一是多了`#ifndef`，二是判断条件改变了。增加`ifndef`是为了提高代码的严谨，防止重复定义`dispatch_main_async_safe`，而关于判断条件改变的的原因则可以参考以下两篇文档：1.[GCD's Main Queue vs. Main Thread](http://blog.benjamin-encz.de/post/main-queue-vs-main-thread/)、2.[Queues are not bound to any specific thread](https://blog.krzyzanowskim.com/2016/06/03/queues-are-not-bound-to-any-specific-thread/)
## 分析：如何判断当前是否在主线程？
### 最简单的方法
&emsp;&emsp;判断当前是否在主线程最简单的方法是使用`[NSThread isMainThread]`，GCD缺少一个类似的方便的API来判断当前是否在主队列上运行，因此一般是使用NSThread的API，如下：
```
if ([NSThread isMainThread]) {
    block();
} else {
    dispatch_async(dispatch_get_main_queue(), block);
}
```
&emsp;&emsp;这在大多数情况下是有效的，直到它出现了异常。下面是关于ReactiveCocoa repo问题的摘录：[ReactiveCocoa issue](https://github.com/ReactiveCocoa/ReactiveCocoa/issues/2635#issuecomment-170215083)
![](http://pic.cloverkim.com/006tKfTcgy1g0e7ys9bihj317m0eagpe.jpg)
&emsp;&emsp;潜在的问题是VektorKit API正在检查是否在主队列上调用它，而不是检查它是否在主线程上运行。虽然每个应用程序都只有一个主线程，但是在这个主线程上执行许多不同的队列是可能的。
&emsp;&emsp;如果库（如VektorKit）依赖于在主队列上检查执行，那么从主线程上执行的非主队列调用API将导致问题。也就是说，如果主线程执行非主队列调度的API，而这个API需要检查是否由主队列上调度，那么将会出现问题。

### 更安全的方法一
&emsp;&emsp;从技术上讲，我认为这是一个`MapKit/VektorKit`漏洞，苹果的UI框架通常保证在从主线程调用时正确工作，没有任何文档提到需要在主队列上执行代码。
&emsp;&emsp;但是，现在我们知道某些api不仅依赖于主线程上的运行，而且还依赖于主队列，因此检查当前队列而不是检查当前线程更安全。
&emsp;&emsp;检查当前队列还可以更好地利用GCD为线程提供的抽象。从技术上讲，我们不应该知道/关心主队列是一种总是绑定到主线程的特殊队列。
&emsp;&emsp;我们需要使用`dispatch_queue_set_specific`函数来将键值与主队列相关联，稍后，我们可以使用`dispatch_queue_get_specific`来检查键和值的存在。
```
- (void)function {
    static void *mainQueueKey = "mainQueueKey";
    dispatch_queue_set_specific(dispatch_get_main_queue(), mainQueueKey, &mainQueueKey, NULL);
    if (dispatch_get_specific(mainQueueKey)) {
        // do something in main queue
        //通过这样判断，就可以真正保证(我们在不主动搞事的情况下)，任务一定是放在主队列中的
    } else {
        // do something in other queue
    }
}
```

### 更安全的方法二（SDWEbImage使用的方法）
&emsp;&emsp;我们知道在使用GCD创建一个queeu的时候回指定queue_label，可以理解为队列名，就像下面：
```
dispatch_queue_t myQueue = dispatch_queue_create("com.apple.threadQueue", DISPATCH_QUEUE_SERIAL);
```
&emsp;&emsp;而第一个参数就是queue_label，根据官方文档解释，这个queue_label是唯一的，所以SDWebImage就采用了这个方式：
```
//取得当前队列的队列名
dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL)
   
//取得主队列的队列名
dispatch_queue_get_label(dispatch_get_main_queue())

然后通过 strcmp 函数进行比较，如果为0 则证明当前队列就是主队列。
```
SDWebImage中的实例：判断当前是否是IOQueue
```
- (void)checkIfQueueIsIOQueue {
    const char *currentQueueLabel = dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL);
    const char *ioQueueLabel = dispatch_queue_get_label(self.ioQueue);
    if (strcmp(currentQueueLabel, ioQueueLabel) != 0) {
        NSLog(@"This method should be called from the ioQueue");
    }
}
```

### 结论
&emsp;&emsp;SDWebImage就是从判断是否在主线程执行改为判断是否由主队列上调度。而由于主队列是一个穿行队列，无论任务是异步同步都不会开辟新线程，所以当前队列是主队列等价于当前在主线程上执行。可以这样说，在主队列调度的任务肯定在主线程执行，而在主线程执行的任务不一定是由主队列调度的。

# SDWebImage的最大并发数和超时时长
&emsp;&emsp;在SDWebImageDownloader类的初始化方法中，对SDWebImage图片下载的最大并发数和超时时长进行了赋值，分别为6和15.0，具体如下所示：
```
// 最大并发数
_downloadQueue.maxConcurrentOperationCount = 6;
// 超时时长
_downloadTimeout = 15.0;
```

# SDWebImage的内存缓存和磁盘缓存是用什么实现的？
## 内存缓存实现——SDMemoryCache
&emsp;&emsp;SDWebImage使用SDMemoryCache（继承自NSCache）来实现内存缓存。NSCache可以设置totalCostLimit来限制缓存的总成本消耗，所以我们再添加缓存的时候需要通过以下代码来指定缓存对象消耗的成本
```
- (void)setObject:(ObjectType)obj forKey:(KeyType)key cost:(NSUInteger)g;
```
&emsp;&emsp;SDMemoryCache在初始化时，会初始化一个弱引用表，当收到内存警告时，会移除内存中缓存的图片，同时保留weakCache，维持对被强引用着的图片的访问。
```
- (instancetype)initWithConfig:(SDImageCacheConfig *)config {
    self = [super init];
    if (self) {
        // 初始化弱引用表，当收到内存警告，内存缓存虽然被清理，但是有些图片已经被其他对象强引用着，.
        // 这时weakCache维持这些图片的弱引用。如果需要获取这些图片就不用去硬盘获取了
        // NSPointerFunctionsStrongMemory 对值进行弱引用，不会对引用计数+1
        self.weakCache = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory capacity:0];
        self.weakCacheLock = dispatch_semaphore_create(1);
        self.config = config;
        // 监听内存警告通知
        [[NSNotificationCenter defaultCenter] addObserver:self
                                                 selector:@selector(didReceiveMemoryWarning:)
                                                     name:UIApplicationDidReceiveMemoryWarningNotification
                                                   object:nil];
    }
    return self;
}

- (void)didReceiveMemoryWarning:(NSNotification *)notification {
    // 当收到内存警告通知，移除内存中缓存的图片，同时保留weakCache，维持对被强引用着的图片的访问
    [super removeAllObjects];
}
```

## 磁盘缓存实现——NSFileManager
&emsp;&emsp;SDImageCache的磁盘缓存是通过异步操作NSFileManager存储缓存文件到沙盒来实现的。

# 读取内存缓存和磁盘缓存的时候如何保证线程安全？
## 读取内存缓存
&emsp;&emsp;NSCache是线程安全的，在多线程操作中，不需要对Cache加锁。读取缓存的时候是在主线程中进行，由于使用NSCache进行存储，所以不需要担心单个value对象的线程安全。

## 读取磁盘缓存
- 创建了一个名为IO的串行队列，所有磁盘操作都在此队列中，逐个执行

    ```
    @property (strong, nonatomic, nullable) dispatch_queue_t ioQueue;
    
    // Create IO serial queue
    _ioQueue = dispatch_queue_create("com.hackemist.SDWebImageCache", DISPATCH_QUEUE_SERIAL);
    ```
    
- 主要存储函数中`dispatch_async(self.ioQueue, ^{})`

    ```
    - (void)storeImage:(nullable UIImage *)image
             imageData:(nullable NSData *)imageData
                forKey:(nullable NSString *)key
                toDisk:(BOOL)toDisk
            completion:(nullable SDWebImageNoParamsBlock)completionBlock {
        // .....
        
        if (toDisk) {
            dispatch_async(self.ioQueue, ^{
                @autoreleasepool {
                    NSData *data = imageData;
                    if (!data && image) {
                        // If we do not have any data to detect image format, check whether it contains alpha channel to use PNG or JPEG format
                        SDImageFormat format;
                        if (SDCGImageRefContainsAlpha(image.CGImage)) {
                            format = SDImageFormatPNG;
                        } else {
                            format = SDImageFormatJPEG;
                        }
                        data = [[SDWebImageCodersManager sharedInstance] encodedDataWithImage:image format:format];
                    }
                    [self _storeImageDataToDisk:data forKey:key];
                }
                
                if (completionBlock) {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        completionBlock();
                    });
                }
            });
        } else {
            if (completionBlock) {
                completionBlock();
            }
        }
    }
    ```
    
## 结论
- 真正的磁盘缓存是在另一个IO专属线程中的一个串行队列下进行的。
- 包括删除、写入等所有磁盘内容都是在这个IO线程中进行，以保证线程安全。
- 但计算大小、获取文件总数等操作，则在主线程中进行，如下代码所示：

    ```
    - (NSUInteger)getSize {
        __block NSUInteger size = 0;
        dispatch_sync(self.ioQueue, ^{
            NSDirectoryEnumerator *fileEnumerator = [self.fileManager enumeratorAtPath:self.diskCachePath];
            for (NSString *fileName in fileEnumerator) {
                NSString *filePath = [self.diskCachePath stringByAppendingPathComponent:fileName];
                NSDictionary<NSString *, id> *attrs = [self.fileManager attributesOfItemAtPath:filePath error:nil];
                size += [attrs fileSize];
            }
        });
        return size;
    }
    ```

# SDWebImage的内存警告是如何处理的？
&emsp;&emsp;利用通知的方式，当收到内存警告的通知（`UIApplicationDidReceiveMemoryWarningNotification`）时，会执行`didReceiveMemoryWarning:`方法，清理内存缓存。

# SDWebImage磁盘缓存的时长是多少？清理操作时间点和清理原则是什么？
## 磁盘缓存的时长默认为一周
```
// SDImageCacheConfig.m
static const NSInteger kDefaultCacheMaxCacheAge = 60 * 60 * 24 * 7; // 1 week
```

## 磁盘清理时间点
分别在『应用被杀死时』和『应用进入后台时』进行清理操作，分别会收到`UIApplicationWillTerminateNotification` 和 `UIApplicationDidEnterBackgroundNotification`两个通知，添加通知的方法如下：

```
[[NSNotificationCenter defaultCenter] addObserver:self
                                         selector:@selector(deleteOldFiles)
                                             name:UIApplicationWillTerminateNotification
                                           object:nil];

[[NSNotificationCenter defaultCenter] addObserver:self
                                         selector:@selector(backgroundDeleteOldFiles)
                                             name:UIApplicationDidEnterBackgroundNotification
                                           object:nil];
```
&emsp;&emsp;当应用程序进入后台时，会涉及到『Long-Running Task』，正常情况下，程序进入后台时，虽然可以继续执行任务，但是在短时间内就会被挂起待机。Long-Running可以让系统为App再多分配一些时间来处理一些耗时任务。
```
- (void)backgroundDeleteOldFiles {
    Class UIApplicationClass = NSClassFromString(@"UIApplication");
    if(!UIApplicationClass || ![UIApplicationClass respondsToSelector:@selector(sharedApplication)]) {
        return;
    }
    UIApplication *application = [UIApplication performSelector:@selector(sharedApplication)];
    // 后台任务标识——注册一个
    __block UIBackgroundTaskIdentifier bgTask = [application beginBackgroundTaskWithExpirationHandler:^{
        // Clean up any unfinished task business by marking where you
        // stopped or ending the task outright.
        [application endBackgroundTask:bgTask];
        bgTask = UIBackgroundTaskInvalid;
    }];

    // 启动long-running任务并立即返回.
    [self deleteOldFilesWithCompletionBlock:^{
        [application endBackgroundTask:bgTask];
        bgTask = UIBackgroundTaskInvalid;
    }];
}
```

## 磁盘清理原则
&emsp;&esmp;清理缓存的规则分两步进行：第一步先清除过期的缓存文件，如果清除掉过期的缓存之后，空间还不够。那么久继续按文件时间从早到晚的排序，先清除最早的缓存文件，直到剩余的空间达到要求。具体的说，SDWebImage是通过以下两个属性控制缓存过期以及剩余空间：
```
// 在缓存中保存图片的最长时间，以秒为单位
@property (assign, nonatomic) NSInteger maxCacheAge;

// 缓存最大大小，以字节为单位
@property (assign, nonatomic) NSUInteger maxCacheSize;
```
对于maxCacheAge和maxCacheSize相关说明：
- maxCacheAge：默认值为1周，单位是秒
- maxCacheSize：没有默认值，意味着默认情况下不会对缓存空间设限制。我们可以通过以下代码进行设置：

    ```
    [SDImageCache sharedImageCache].maxCacheSize = 1024 * 1024 * 50;    // 50M
    ```
    maxCacheSize是以字节来表示的，上面的代码中，设置的是50M的最大缓存空间，把maxCacheSize的设置代码写在App启动的时候，这样SDWebImage在清理缓存的时候，就会清理多余的缓存文件了。
    
# SDWebImage 磁盘目录位于哪里？
- 缓存在沙盒目录下`Library/Caches`，默认情况下，二级目录为`~/Library/Caches/default/com.hackemist.SDWebImageCache.default`
- 也可以自定义缓存文件名，相关代码（SDImageCache）如下： 

    ```
    - (instancetype)init {
        return [self initWithNamespace:@"default"];
    }
    
    - (nonnull instancetype)initWithNamespace:(nonnull NSString *)ns {
        NSString *path = [self makeDiskCachePath:ns];
        return [self initWithNamespace:ns diskCacheDirectory:path];
    }
    
    - (nonnull instancetype)initWithNamespace:(nonnull NSString *)ns
                           diskCacheDirectory:(nonnull NSString *)directory {
        if ((self = [super init])) {
            NSString *fullNamespace = [@"com.hackemist.SDWebImageCache." stringByAppendingString:ns];
            
            // Create IO serial queue
            _ioQueue = dispatch_queue_create("com.hackemist.SDWebImageCache", DISPATCH_QUEUE_SERIAL);
            
            _config = [[SDImageCacheConfig alloc] init];
            
            // Init the memory cache
            _memCache = [[SDMemoryCache alloc] initWithConfig:_config];
            _memCache.name = fullNamespace;
    
            // .....
    }
    ```
    
# 下载图片的URL必须是NSURL吗？
&emsp;&emsp;不是必须传NSURL，在SDWebImageManager中有容错处理。所以即使传入一个字符串依旧可以正确的下载图片，相关代码如下：
```
if ([url isKindOfClass:NSString.class]) {
    url = [NSURL URLWithString:(NSString *)url];
}

// Prevents app crashing on argument type error like sending NSNull instead of NSURL
if (![url isKindOfClass:NSURL.class]) {
    url = nil;
}
```

# 参考
- [简书 - lionsom_lin - SDWebImage4.0源码探究（一）面试题](https://www.jianshu.com/p/b8517dc833c7)
- [cocoachina - yyuuzhu - iOS源码补完计划--SDWebImage4.0+源码参阅(附面试题/流程图)](http://www.cocoachina.com/ios/20180413/23006.html)
- [Swift Cafe - swift - 天天都在用的 SDWebImage， 你了解它的缓存策略吗？](http://swiftcafe.io/2017/02/19/sdimage-cache/)
- [简书 - ShannonChenCHN - SDWebImage 源码阅读笔记](https://www.jianshu.com/p/06f0265c22eb#)