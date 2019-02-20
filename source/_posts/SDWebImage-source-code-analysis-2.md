---
title: SDWebImage源码解析（二）
categories: iOS开发
tags:
- iOS
- 源码解析
top: 100
copyright: ture
---

# 概述
&emsp;&emsp;在上一篇文章[SDWebImage源码解析（一）](http://cloverkim.com/SDWebImage-source-code-analysis-1.html)中，主要介绍了SDWebImage的基本架构、各种类的作用、相关工具类（NSData+ImageContentType、SDWebImageCodersManager、SDWebImageImageIOCoder以及SDWebImageCompat）的详细分析。在本文中，将会分析SDWebImage的UIView+WebCache、UIWebImageManager相关核心类。<!-- more -->

# 详细分析
## 核心类分析
### UIView+WebCache
&emsp;&emsp;在UIImageView+WebCache和UIButton+WebCache等分类中，提供了以下对外使用的方法：
```
- (void)sd_setImageWithURL:(nullable NSURL *)url NS_REFINED_FOR_SWIFT;
- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder;
- (void)sd_setImageWithURL:(nullable NSURL *)url
          placeholderImage:(nullable UIImage *)placeholder
                   options:(SDWebImageOptions)options
```
&emsp;&emsp;从上面分类的接口中，可以看出其体现了设计模式五大原则之一的接口分离原则，那何为接口分离分则：**为特定功能提供特定的接口，不要使用单一的总接口包括所有功能，而是应该根据功能把这些接口分割，减少依赖，不能强迫用户去依赖那些他们不使用的接口。**当调用了上面几个接口时，又会调用UIView+WebCache分类的`sd_internalSetImageWithURL`方法来做图片加载请求。具体是通过SDWebImageManager调用来实现的。同时实现了Operation取消、ActivityIndicator的添加和取消。下面先来看下`sd_internalSetImageWithURL`方法的实现：
```
- (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                  placeholderImage:(nullable UIImage *)placeholder
                           options:(SDWebImageOptions)options
                      operationKey:(nullable NSString *)operationKey
                     setImageBlock:(nullable SDSetImageBlock)setImageBlock
                          progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                         completed:(nullable SDExternalCompletionBlock)completedBlock
                           context:(nullable NSDictionary<NSString *, id> *)context {
    // 根据参数operationKey取消当前类所对应的下载Operation对象，如果operationKey为nil，则key取NSStringFromClass([self class])
    NSString *validOperationKey = operationKey ?: NSStringFromClass([self class]);
    // 具体的取消操作在UIView+WebCacheOperation中实现
    [self sd_cancelImageLoadOperationWithKey:validOperationKey];
    // 利用关联对象给当前self实例绑定url，key为imageURLKey，value为url
    objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
    // 利用与运算判断调用者是否需要设置占位图
    if (!(options & SDWebImageDelayPlaceholder)) {
        if ([context valueForKey:SDWebImageInternalSetImageGroupKey]) {
            dispatch_group_t group = [context valueForKey:SDWebImageInternalSetImageGroupKey];
            dispatch_group_enter(group);
        }
        dispatch_main_async_safe(^{
            [self sd_setImage:placeholder imageData:nil basedOnClassOrViaCustomSetImageBlock:setImageBlock];
        });
    }
    
    if (url) {
        // 判断之前是否利用关联对象给self添加了显示loading加载
        if ([self sd_showActivityIndicatorView]) {
            [self sd_addActivityIndicator];
        }
        
        // reset the progress
        self.sd_imageProgress.totalUnitCount = 0;
        self.sd_imageProgress.completedUnitCount = 0;
        
        SDWebImageManager *manager;
        if ([context valueForKey:SDWebImageExternalCustomManagerKey]) {
            manager = (SDWebImageManager *)[context valueForKey:SDWebImageExternalCustomManagerKey];
        } else {
            manager = [SDWebImageManager sharedManager];
        }
        
        __weak __typeof(self)wself = self;
        // 图片下载进度block回调
        SDWebImageDownloaderProgressBlock combinedProgressBlock = ^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
            wself.sd_imageProgress.totalUnitCount = expectedSize;
            wself.sd_imageProgress.completedUnitCount = receivedSize;
            if (progressBlock) {
                progressBlock(receivedSize, expectedSize, targetURL);
            }
        };
        // 调用SDWebImageManager的loadImageWithURL方法加载图片，返回值是SDWebImageCombinedOperation
        id <SDWebImageOperation> operation = [manager loadImageWithURL:url options:options progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
            __strong __typeof (wself) sself = wself;
            if (!sself) { return; }
            // 移除loading加载
            [sself sd_removeActivityIndicator];
            // 如果没有更新进度，则将其标记为完成状态
            if (finished && !error && sself.sd_imageProgress.totalUnitCount == 0 && sself.sd_imageProgress.completedUnitCount == 0) {
                sself.sd_imageProgress.totalUnitCount = SDWebImageProgressUnitCountUnknown;
                sself.sd_imageProgress.completedUnitCount = SDWebImageProgressUnitCountUnknown;
            }
            BOOL shouldCallCompletedBlock = finished || (options & SDWebImageAvoidAutoSetImage);
            BOOL shouldNotSetImage = ((image && (options & SDWebImageAvoidAutoSetImage)) ||
                                      (!image && !(options & SDWebImageDelayPlaceholder)));
            SDWebImageNoParamsBlock callCompletedBlockClojure = ^{
                if (!sself) { return; }
                if (!shouldNotSetImage) {
                    [sself sd_setNeedsLayout];
                }
                if (completedBlock && shouldCallCompletedBlock) {
                    completedBlock(image, error, cacheType, url);
                }
            };
            
            // case 1a: we got an image, but the SDWebImageAvoidAutoSetImage flag is set
            // OR
            // case 1b: we got no image and the SDWebImageDelayPlaceholder is not set
            // 如果设置了不自动显示图片，则回调让调用者手动添加显示图片
            if (shouldNotSetImage) {
                dispatch_main_async_safe(callCompletedBlockClojure);
                return;
            }
            
            UIImage *targetImage = nil;
            NSData *targetData = nil;
            if (image) {
                // case 2a: we got an image and the SDWebImageAvoidAutoSetImage is not set
                targetImage = image;
                targetData = data;
            } else if (options & SDWebImageDelayPlaceholder) {
                // case 2b: we got no image and the SDWebImageDelayPlaceholder flag is set
                targetImage = placeholder;
                targetData = nil;
            }
            
            // check whether we should use the image transition
            SDWebImageTransition *transition = nil;
            if (finished && (options & SDWebImageForceTransition || cacheType == SDImageCacheTypeNone)) {
                transition = sself.sd_imageTransition;
            }
            if ([context valueForKey:SDWebImageInternalSetImageGroupKey]) {
                dispatch_group_t group = [context valueForKey:SDWebImageInternalSetImageGroupKey];
                dispatch_group_enter(group);
                dispatch_main_async_safe(^{
                    [sself sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock transition:transition cacheType:cacheType imageURL:imageURL];
                });
                // ensure completion block is called after custom setImage process finish
                dispatch_group_notify(group, dispatch_get_main_queue(), ^{
                    callCompletedBlockClojure();
                });
            } else {
                dispatch_main_async_safe(^{
                    [sself sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock transition:transition cacheType:cacheType imageURL:imageURL];
                    callCompletedBlockClojure();
                });
            }
        }];
        // 绑定operation到当前self，key为validOperationKey，value为operation
        [self sd_setImageLoadOperation:operation forKey:validOperationKey];
    } else {
        // 移除loading加载框，并抛出异常的回调
        dispatch_main_async_safe(^{
            [self sd_removeActivityIndicator];
            if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:-1 userInfo:@{NSLocalizedDescriptionKey : @"Trying to load a nil url"}];
                completedBlock(nil, error, SDImageCacheTypeNone, url);
            }
        });
    }
}
```
&emsp;&emsp;`UIView+WebCache`是通过`UIView+WebCacheOperation`分类来实现UIView的图片下载Operation的关联和取消。
```
/// 关联operation对象与key对象
- (void)sd_setImageLoadOperation:(nullable id<SDWebImageOperation>)operation forKey:(nullable NSString *)key {
    if (key) {
        [self sd_cancelImageLoadOperationWithKey:key];
        if (operation) {
            SDOperationsDictionary *operationDictionary = [self sd_operationDictionary];
            @synchronized (self) {
                [operationDictionary setObject:operation forKey:key];
            }
        }
    }
}

/// 取消当前key对应的所有实现了SDWebImageOperation协议的Operation对象
- (void)sd_cancelImageLoadOperationWithKey:(nullable NSString *)key {
    // 获取当前view对应的所有key
    SDOperationsDictionary *operationDictionary = [self sd_operationDictionary];
    id<SDWebImageOperation> operation;
    // 获取对应的图片加载operation
    @synchronized (self) {
        operation = [operationDictionary objectForKey:key];
    }
    // 取消所有当前view对应的所有operation
    if (operation) {
        if ([operation conformsToProtocol:@protocol(SDWebImageOperation)]){
            [operation cancel];
        }
        @synchronized (self) {
            [operationDictionary removeObjectForKey:key];
        }
    }
}

/// 移除operation对象与key对象的关联
- (void)sd_removeImageLoadOperationWithKey:(nullable NSString *)key {
    if (key) {
        SDOperationsDictionary *operationDictionary = [self sd_operationDictionary];
        @synchronized (self) {
            [operationDictionary removeObjectForKey:key];
        }
    }
}
```

### UIWebImageManager
&emsp;&emsp;UIWebImageManager类是SDWebImage中的核心类，主要负责调用SDWebImageDownloader进行图片下载，以及在下载完之后利用SDImageCache进行图片的缓存。UIImageView等各种视图都是通过UIView+WebCache分类的`sd_internalSetImageWithURL`方法来调用SDWebImageManager类的如下方法来实现图片加载：
```
/// 这个方法为核心方法，UIImageView等分类都默认通过调用这个方法来获取数据
- (nullable id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                              options:(SDWebImageOptions)options
                                             progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                            completed:(nullable SDInternalCompletionBlock)completedBlock;
```
&emsp;&emsp;在上面的方法中，第二个参数options，这个参数指定了图片加载过程中的不同选项，SDWebImage可以根据不同的选项做不同的处理，这一个枚举类型，具体如下：
```
typedef NS_OPTIONS(NSUInteger, SDWebImageOptions) {
    /**
     * 默认情况下，当URL无法下载时，该URL将被列入黑名单，下次再有该URL的请求时，将不会继续尝试.
     * 如果为true，则禁用黑名单，表示需要再尝试请求.
     */
    SDWebImageRetryFailed = 1 << 0,

    /**
     * 默认情况下，图片加载是在交互期间启动的，这个标志可以禁用这个时候加载,
     * 而是当UIScrollView开始减速滑动的时候开始加载.
     */
    SDWebImageLowPriority = 1 << 1,

    /**
     * 此标志在下载完成后禁止磁盘上缓存，仅在内存中缓存
     */
    SDWebImageCacheMemoryOnly = 1 << 2,

    /**
     * 该标记支持渐进式下载，在下载过程中会像浏览器那样逐步显示图像.
     * 默认情况下，图片只会在加载完成以后再显示.
     */
    SDWebImageProgressiveDownload = 1 << 3,

    /**
     * 即使本地已经缓存了图片，但是也要根据HTTP的缓存策略去网上加载图片.
     * 磁盘缓存将由NSURLCache来处理，而不是SDWebImage，这会导致性能轻微下降.
     * 这个选项专门用于处理，url地址没有变，但是url对应的图片数据在服务器改变的情况.
     * 如果一个缓存图片更新了，则completion这个回调会被调用两次，一次返回缓存图片，一次返回最终图片.
     *
     * 只有在不能确保URL和他对应的内容不能完全对应的时候才使用此标志.
     */
    SDWebImageRefreshCached = 1 << 4,

    /**
     * 在iOS 4+中，如果应用进入后台，继续下载图片。这是通过请求系统来实现的
     * 额外的后台时间让请求完成。如果后台任务超时，操作将被取消.
     */
    SDWebImageContinueInBackground = 1 << 5,

    /**
     * 处理储存在NSHTTPCookieStore中的cookie，通过设置NSMutableURLRequest.HTTPShouldHandleCookies = YES来实现的
     */
    SDWebImageHandleCookies = 1 << 6,

    /**
     * 启用不受信任的SSL证书.
     * 用于测试目的，但在生产环境中小心使用
     */
    SDWebImageAllowInvalidSSLCertificates = 1 << 7,

    /**
     * 默认情况下，图片加载的顺序是根据假如队列的顺序加载的。但是这个标记会把任务添加到队列的最前面.
     */
    SDWebImageHighPriority = 1 << 8,
    
    /**
     * 默认情况下，在加载图片时会加载占位图。
     * 此标记会阻止显示占位图直到图片加载完成.
     */
    SDWebImageDelayPlaceholder = 1 << 9,

    /**
     * 我们通常不会再动画图像上调用transformDownloadedImage代理方法,
     * 因为大部分transformation操作会对图片做无用处理.
     * 用这个标记表示无论如何都要对图片做transform处理.
     */
    SDWebImageTransformAnimatedImage = 1 << 10,
    
    /**
     * 默认情况下，图片在下载后被添加到imageView上。但在某些情况下，我们希望
     * UIImageView加载我们手动处理之后的图片
     * 这个标记允许我们在completion这个Block中手动设置处理好以后的图片。
     */
    SDWebImageAvoidAutoSetImage = 1 << 11,
    
    /**
     * 默认情况下，图片会根据其原始大小进行解码。在iOS上，这个标记会将图片缩小到与设备受限内存兼容的大小
     * 如果 `SDWebImageProgressiveDownload` 标记被设置了，则这个flag不起作用.
     */
    SDWebImageScaleDownLargeImages = 1 << 12,
    
    /**
     * 默认情况下，当图片缓存在内存中时，我们不会查询磁盘数据。这个掩码可以强制同时查询磁盘数据.
     * 建议将此标记与 `SDWebImageQueryDiskSync` 一起使用，以确保在相同的runloop中加载图片.
     */
    SDWebImageQueryDataWhenInMemory = 1 << 13,
    
    /**
     * 默认情况下，我们同步查询内存缓存，异步查询磁盘缓存。这个掩码可以强制同步查询磁盘缓存，以确保在相同的runloop循环中加载图片.
     */
    SDWebImageQueryDiskSync = 1 << 14,
    
    /**
     * 默认情况下，当缓存丢失时，会从网络下载图片。此标记仅能防止网络从缓存加载.
     */
    SDWebImageFromCacheOnly = 1 << 15,
    /**
     * 默认情况下，当使用`SDWebImageTransition`在图片加载完成后执行一些视图转换时，此转换仅适用于从网络下载的图片。这个掩码还可以强制将视图转换应用于内存和磁盘缓存.
     */
    SDWebImageForceTransition = 1 << 16
};
```
接下来看下SDWebImageManager的初始化过程：
```
/// 单例
+ (nonnull instancetype)sharedManager {
    static dispatch_once_t once;
    static id instance;
    dispatch_once(&once, ^{
        instance = [self new];
    });
    return instance;
}

- (nonnull instancetype)init {
    SDImageCache *cache = [SDImageCache sharedImageCache];
    SDWebImageDownloader *downloader = [SDWebImageDownloader sharedDownloader];
    return [self initWithCache:cache downloader:downloader];
}

/// 初始化SDImageCache和SDWebImageDownloader对象
- (nonnull instancetype)initWithCache:(nonnull SDImageCache *)cache downloader:(nonnull SDWebImageDownloader *)downloader {
    if ((self = [super init])) {
        _imageCache = cache;
        _imageDownloader = downloader;
        // 用于保存加载失败的url集合
        _failedURLs = [NSMutableSet new];
        // 用于保存当前正在加载的Operation
        _runningOperations = [NSMutableArray new];
    }
    return self;
}
```
然后是SDWebImageManager的核心方法`loadImageWithURL`，其实现过程如下：
```
- (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                     options:(SDWebImageOptions)options
                                    progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                   completed:(nullable SDInternalCompletionBlock)completedBlock {
    // 利用NSAssert预处理宏，进行判断参数completedBlock，如果为nil，则抛出异常，反之继续执行
    NSAssert(completedBlock != nil, @"If you mean to prefetch the image, use -[SDWebImagePrefetcher prefetchURLs] instead");

    // 如果传入的url是NSString类型的，则转换为NSURL类型再进行处理.
    if ([url isKindOfClass:NSString.class]) {
        url = [NSURL URLWithString:(NSString *)url];
    }

    // 如果url不是NSURL类型的对象，则将其置为nil
    if (![url isKindOfClass:NSURL.class]) {
        url = nil;
    }

    // 图片加载获取过程中绑定一个 SDWebImageCombinedOperation 对象，方便接下来再通过找个对象对url的加载控制
    SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
    operation.manager = self;

    BOOL isFailedUrl = NO;
    // 判断url是否在加载失败的url集合里面
    if (url) {
        @synchronized (self.failedURLs) {
            isFailedUrl = [self.failedURLs containsObject:url];
        }
    }

    // 如果url是失败的url或者有其他错误时，则直接根据operation来做异常情况处理
    if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
        [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorFileDoesNotExist userInfo:nil] url:url];
        return operation;
    }

    // 把加载图片的一个载体存入runningOperations。里面是所有正在做图片加载过程的operation的集合    
    @synchronized (self.runningOperations) {
        [self.runningOperations addObject:operation];
    }
    // 根据url获取url对应的key
    NSString *key = [self cacheKeyForURL:url];
    
    SDImageCacheOptions cacheOptions = 0;
    if (options & SDWebImageQueryDataWhenInMemory) cacheOptions |= SDImageCacheQueryDataWhenInMemory;
    if (options & SDWebImageQueryDiskSync) cacheOptions |= SDImageCacheQueryDiskSync;
    
    // 如果图片是从内存缓存加载，则返回的cacheOperation为nil
    // 如果是从磁盘加载，则返回的cacheOperation是NSOperation对象
    // 如果是从网络加载，则返回的cacheOperation对象是 SDWebImageDownloaderOperation 对象
    __weak SDWebImageCombinedOperation *weakOperation = operation;
    operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:^(UIImage *cachedImage, NSData *cachedData, SDImageCacheType cacheType) {
        __strong __typeof(weakOperation) strongOperation = weakOperation;
        // 如果已经取消了操作，则直接返回并且移除对应的operation对象
        if (!strongOperation || strongOperation.isCancelled) {
            [self safelyRemoveOperationFromRunning:strongOperation];
            return;
        }
        
        // 检查判断是否需要从网络上下载图片
        BOOL shouldDownload = (!(options & SDWebImageFromCacheOnly))
            && (!cachedImage || options & SDWebImageRefreshCached)
            && (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url]);
        if (shouldDownload) {
            if (cachedImage && options & SDWebImageRefreshCached) {
                // 如果在缓存中获取到了图片，但设置了SDWebImageRefreshCached来忽略缓存，则通知缓存的图片并尝试重新下载它.
                [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
            }

            // 如果没有图片或者请求刷新，则下载图片
            // 把图片加载的 SDWebImageOperations 类型枚举转换为图片下载的 SDWebImageDownloaderOptions 类型的枚举
            SDWebImageDownloaderOptions downloaderOptions = 0;
            if (options & SDWebImageLowPriority) downloaderOptions |= SDWebImageDownloaderLowPriority;
            if (options & SDWebImageProgressiveDownload) downloaderOptions |= SDWebImageDownloaderProgressiveDownload;
            if (options & SDWebImageRefreshCached) downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
            if (options & SDWebImageContinueInBackground) downloaderOptions |= SDWebImageDownloaderContinueInBackground;
            if (options & SDWebImageHandleCookies) downloaderOptions |= SDWebImageDownloaderHandleCookies;
            if (options & SDWebImageAllowInvalidSSLCertificates) downloaderOptions |= SDWebImageDownloaderAllowInvalidSSLCertificates;
            if (options & SDWebImageHighPriority) downloaderOptions |= SDWebImageDownloaderHighPriority;
            if (options & SDWebImageScaleDownLargeImages) downloaderOptions |= SDWebImageDownloaderScaleDownLargeImages;
            
            // 如果设置了强制刷新缓存的选项，则 SDWebImageDownloaderProgressiveDownload 选项失效并且添加 SDWebImageDownloaderIgnoreCachedResponse 选项
            if (cachedImage && options & SDWebImageRefreshCached) {
                // 如果图片已经缓存但强制刷新，则强制关闭渐进
                downloaderOptions &= ~SDWebImageDownloaderProgressiveDownload;
                // 忽略从NSURLCache获取的图片（如果有缓存），但需要强制刷新
                downloaderOptions |= SDWebImageDownloaderIgnoreCachedResponse;
            }
            
            // `SDWebImageCombinedOperation` -> `SDWebImageDownloadToken` -> `downloadOperationCancelToken`, which is a `SDCallbacksDictionary` and retain the completed block below, so we need weak-strong again to avoid retain cycle
            __weak typeof(strongOperation) weakSubOperation = strongOperation;
            strongOperation.downloadToken = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *downloadedData, NSError *error, BOOL finished) {
                __strong typeof(weakSubOperation) strongSubOperation = weakSubOperation;
                // 如果图片下载结束以后，对应的图片加载操作已经取消，则不做任何处理
                if (!strongSubOperation || strongSubOperation.isCancelled) {
                    // Do nothing if the operation was cancelled
                    // See #699 for more details
                    // if we would call the completedBlock, there could be a race condition between this block and another completedBlock for the same object, so if this one is called second, we will overwrite the new data
                } else if (error) {
                    // 如果加载出错，则直接返回回调，并且添加failedURLs集合中
                    [self callCompletionBlockForOperation:strongSubOperation completion:completedBlock error:error url:url];
                    BOOL shouldBlockFailedURL;
                    // Check whether we should block failed url
                    if ([self.delegate respondsToSelector:@selector(imageManager:shouldBlockFailedURL:withError:)]) {
                        shouldBlockFailedURL = [self.delegate imageManager:self shouldBlockFailedURL:url withError:error];
                    } else {
                        shouldBlockFailedURL = (   error.code != NSURLErrorNotConnectedToInternet
                                                && error.code != NSURLErrorCancelled
                                                && error.code != NSURLErrorTimedOut
                                                && error.code != NSURLErrorInternationalRoamingOff
                                                && error.code != NSURLErrorDataNotAllowed
                                                && error.code != NSURLErrorCannotFindHost
                                                && error.code != NSURLErrorCannotConnectToHost
                                                && error.code != NSURLErrorNetworkConnectionLost);
                    }
                    
                    if (shouldBlockFailedURL) {
                        @synchronized (self.failedURLs) {
                            [self.failedURLs addObject:url];
                        }
                    }
                }
                else {       // 网络图片加载成功
                    // 如果有重试失败下载的选项，则把url从failedURLs集合中移除
                    if ((options & SDWebImageRetryFailed)) {
                        @synchronized (self.failedURLs) {
                            [self.failedURLs removeObject:url];
                        }
                    }
                    
                    BOOL cacheOnDisk = !(options & SDWebImageCacheMemoryOnly);
                    
                    // We've done the scale process in SDWebImageDownloader with the shared manager, this is used for custom manager and avoid extra scale.
                    if (self != [SDWebImageManager sharedManager] && self.cacheKeyFilter && downloadedImage) {
                        downloadedImage = [self scaledImageForKey:key image:downloadedImage];
                    }

                    if (options & SDWebImageRefreshCached && cachedImage && !downloadedImage) {
                        // Image refresh hit the NSURLCache cache, do not call the completion block
                    } else if (downloadedImage && (!downloadedImage.images || (options & SDWebImageTransformAnimatedImage)) && [self.delegate respondsToSelector:@selector(imageManager:transformDownloadedImage:withURL:)]) {
                        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                            // 获取transform后的图片
                            UIImage *transformedImage = [self.delegate imageManager:self transformDownloadedImage:downloadedImage withURL:url];

                            // 存储transform以后的图片
                            if (transformedImage && finished) {
                                BOOL imageWasTransformed = ![transformedImage isEqual:downloadedImage];
                                NSData *cacheData;
                                // pass nil if the image was transformed, so we can recalculate the data from the image
                                if (self.cacheSerializer) {
                                    cacheData = self.cacheSerializer(transformedImage, (imageWasTransformed ? nil : downloadedData), url);
                                } else {
                                    cacheData = (imageWasTransformed ? nil : downloadedData);
                                }
                                [self.imageCache storeImage:transformedImage imageData:cacheData forKey:key toDisk:cacheOnDisk completion:nil];
                            }
                            
                            // 回调拼接
                            [self callCompletionBlockForOperation:strongSubOperation completion:completedBlock image:transformedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
                        });
                    } else {
                        // 下载完成并且有image，则缓存图片
                        if (downloadedImage && finished) {
                            if (self.cacheSerializer) {
                                dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                                    NSData *cacheData = self.cacheSerializer(downloadedImage, downloadedData, url);
                                    [self.imageCache storeImage:downloadedImage imageData:cacheData forKey:key toDisk:cacheOnDisk completion:nil];
                                });
                            } else {
                                [self.imageCache storeImage:downloadedImage imageData:downloadedData forKey:key toDisk:cacheOnDisk completion:nil];
                            }
                        }
                        [self callCompletionBlockForOperation:strongSubOperation completion:completedBlock image:downloadedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
                    }
                }

                // 如果下载和缓存都完成了，则删除操作队列中的operation
                if (finished) {
                    [self safelyRemoveOperationFromRunning:strongSubOperation];
                }
            }];
        } else if (cachedImage) {
            // 有图片且线程没有被取消，则回调有图片的completeBlock
            [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
            [self safelyRemoveOperationFromRunning:strongOperation];
        } else {
            // 图片没有在缓存中，并且代理方法也不允许下载则回调失败
            [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:nil data:nil error:nil cacheType:SDImageCacheTypeNone finished:YES url:url];
            [self safelyRemoveOperationFromRunning:strongOperation];
    }];

    return operation;
}
```
通过对`loadImageWithURL`方法的分析，主要实现的功能有：
- 创建一个`SDWebImageCombinedOperation`对象，调用者可以通过这个对象来对加载做取消等操作，这个对象的cancelOperation属性有以下几种情况：
 - 如果图片是从内存加载，则返回的cacheOperation为nil
 - 如果是从磁盘加载，则返回的cacheOperation是NSOperation对象
 - 如果是从网络加载，则返回的cacheOperation是SDWebImageDownloaderOperation对象

- 通过failedURLs集合存储加载失败的url，防止失败的url再次加载
- 通过runningOperations来存储正在执行下载图片的操作
- 图片加载结束以后，通过`callCompletionBlockForOperation`方法来拼接回调Block
- 图片成功从网络下载以后，通过imageCache的`storeImage`方法来缓存图片

SDWebImageManager类其它方法如下：
```
/// 根据URL获取缓存中的key
- (nullable NSString *)cacheKeyForURL:(nullable NSURL *)url {
    if (!url) {
        return @"";
    }

    if (self.cacheKeyFilter) {
        return self.cacheKeyFilter(url);
    } else {
        return url.absoluteString;
    }
}

/// 检查缓存中是否缓存了当前url对应的图片，先判断内存缓存，再判断磁盘缓存
- (void)cachedImageExistsForURL:(nullable NSURL *)url
                     completion:(nullable SDWebImageCheckCacheCompletionBlock)completionBlock {
    NSString *key = [self cacheKeyForURL:url];
    // 判断是否存在内存缓存
    BOOL isInMemoryCache = ([self.imageCache imageFromMemoryCacheForKey:key] != nil);
    
    if (isInMemoryCache) {
        // making sure we call the completion block on the main queue
        dispatch_async(dispatch_get_main_queue(), ^{
            if (completionBlock) {
                completionBlock(YES);
            }
        });
        return;
    }
    // 判断是否存在磁盘缓存
    [self.imageCache diskImageExistsWithKey:key completion:^(BOOL isInDiskCache) {
        // the completion block of checkDiskCacheForImageWithKey:completion: is always called on the main queue, no need to further dispatch
        if (completionBlock) {
            completionBlock(isInDiskCache);
        }
    }];
}

/// 根据url判断磁盘中是否存在图片 
- (void)diskImageExistsForURL:(nullable NSURL *)url
                   completion:(nullable SDWebImageCheckCacheCompletionBlock)completionBlock {
    NSString *key = [self cacheKeyForURL:url];
    
    [self.imageCache diskImageExistsWithKey:key completion:^(BOOL isInDiskCache) {
        // the completion block of checkDiskCacheForImageWithKey:completion: is always called on the main queue, no need to further dispatch
        if (completionBlock) {
            completionBlock(isInDiskCache);
        }
    }];
}


/// 将图片存入缓存中
- (void)saveImageToCache:(nullable UIImage *)image forURL:(nullable NSURL *)url {
    if (image && url) {
        NSString *key = [self cacheKeyForURL:url];
        [self.imageCache storeImage:image forKey:key toDisk:YES completion:nil];
    }
}


/// 取消所有下载操作
- (void)cancelAll {
    @synchronized (self.runningOperations) {
        NSArray<SDWebImageCombinedOperation *> *copiedOperations = [self.runningOperations copy];
        [copiedOperations makeObjectsPerformSelector:@selector(cancel)];
        [self.runningOperations removeObjectsInArray:copiedOperations];
    }
}

/// 判断当前是否有图片正在下载
- (BOOL)isRunning {
    BOOL isRunning = NO;
    @synchronized (self.runningOperations) {
        isRunning = (self.runningOperations.count > 0);
    }
    return isRunning;
}

/// 线程安全地移除下载operation
- (void)safelyRemoveOperationFromRunning:(nullable SDWebImageCombinedOperation*)operation {
    @synchronized (self.runningOperations) {
        if (operation) {
            [self.runningOperations removeObject:operation];
        }
    }
}
```

# 总结
&emsp;&emsp;本文中，主要详细分析了SDWebImage的UIView+WebCache、UIWebImageManager两个核心类。在下文中[SDWebImage源码解析（三）](http://cloverkim.com/SDWebImage-source-code-analysis-3.html)，将会继续分析SDWebImage的相关核心类。