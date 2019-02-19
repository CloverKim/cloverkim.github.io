---
title: SDWebImage源码解析（一）
categories: iOS开发
tags:
- iOS
- 源码解析
top: 100
copyright: ture
---

&emsp;&emsp;[SDWebImage](https://github.com/SDWebImage/SDWebImage)是我们常用的图片缓存加载库，基本是iOS项目中标配的第三方库。SDWebImage提供了图片从加载、解析、处理、缓存、清理等一系列功能。对于一个我们常用的库，就很有必要对源码进行仔细阅读与学习，以便了解更多SDWebImage支持的功能与实现原理，当然学习和分析优秀的源码，也能为我们在调试解决问题时提供一定帮助。<!-- more -->
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g05syshw7tj30ct0360so.jpg 'SDWebImage')

# 基本架构图
下图为作者给出的SDWebImage基本架构图：
![](https://ws4.sinaimg.cn/large/006tKfTcgy1g05wka6jb7j314q0u0dox.jpg '基本架构图')
由上图架构图，我们将SDWebImage的相关类分为以下三种：
- 各种分类：
    - UIButton+WebCache：为UIButton类添加加载图片的方法。
    - MKAnnotationView+WebCache：为MKAnnotationView类添加各种加载图片的方法。
    - UIImageView+WebCache：为UIImageView类添加加载图片的方法。
    - UIImageView+HighlightedWebCache：为UIImageView类添加高亮状态下加载图片的方法。
- 工具类：
    - NSData+ImageContentType：根据图片数据获取图片的类型，比如GIF、PNG等。
    - UIImage+MultiFormat：根据UIImage的data生成指定格式的UIImage。
    - UIImage+GIF：判断一张图是否为GIF。
    - SDWebImageCompat：根据屏幕的分辨倍数成倍放大或者缩小图片的大小。
    - SDImageCacheConfig：图片缓存策略记录。比如是否解压缩、是否允许iCloud、是否允许内存缓存、缓存时间等。
    - SDWebImageCodersManager：编码解码管理器，处理多个图片编码解码任务，编码器是一个优先队列，这意味着后面添加的编码器将具有最高优先级。
    
- 核心类：
    - UIView+WebCache：所有的UIView及其子类都会调用这个分类的方法来完成图片加载的处理，同时通过UIView+WebCacheOperation分类来管理请求的取消和记录工作。
    - SDImageCache：负责SDWebImage的整个缓存工作，是一个单例对象。缓存路径处理、缓存名字处理、管理内存缓存和磁盘缓存的创建和删除、根据指定key获取图片、存入图片的处理、根据缓存的创建和修改日期来删除缓存等。
    - SDWebImageManager：拥有一个SDImageCache和SDWebImageDownloader属性，分别用于图片的缓存和加载处理。为UIView及其子类提供了加载图片的统一接口。
    - SDWebImageDownloader：图片下载中心，管理下载队列。
    - SDWebImageDownloaderOperation：用于下载图片，管理NSURLRequest对象请求头的封装、缓存、cookie的设置、加载选项的处理等。

当然，SDWebImage框架所包含的类不仅仅是上面罗列的，具体的相关类可以阅读源码。

# 流程
下图为SDWebImage图片加载的时序图，实现了图片加载、数据处理、图片缓存等一系列工作。
![](https://ws2.sinaimg.cn/large/006tKfTcgy1g05xvrthcvj321m0r4q68.jpg 'SDWebImage图片加载时序图')
由上图时序图，SDWebImage加载图片的流程大致如下：
1. 对象调用暴露的接口方法` sd_setImageWithURL() `时，会再调用` setImageWithURL:placeholderImage:options: `方法，先把占位图placeholderImage显示，然后SDWebImageManager根据URL开始处理图片。
2. SDImageCache类先从内存缓存查找是否有图片缓存，如果内存中已经有图片缓存，则直接回调到前端进行图片的显示。
3. 如果内存缓存中没有，则生成NSInvocationOperation添加到队列开始从硬盘中查找图片是否已经缓存。根据url为key在硬盘缓存目录下尝试读取图片文件，这一步是在NSOperation下进行的操作，所以需要回到主线程进行查找结果的回调。如果从硬盘读取到了图片，则将图片添加到内存缓存中，然后再回调到前端进行图片的显示。如果从硬盘缓存目录读取不到图片，说明所有缓存都不存在该图片，则需要下载图片。
4. 共享或重新生成一个下载器SDWebImageDownloader开始下载图片。图片的下载由NSURLConnection来处理，实现相关delegate来判断的下载状态：下载中、下载完成和下载失败。
5. 图片数据下载完成之后，交给SDWebImageDecoder类做图片解码处理，图片的解码处理在NSOperationQueue完成，不会阻塞主线程。在图片解码完成后，会回调给SDWebImageDownloader，然后回调给SDWebImageManager告知图片下载完成，通知所有的downloadDelegates下载完成，回调给需要的地方显示图片。
6. 最后将图片通过SDImageCache类，同时保存到内存缓存和硬盘缓存中。写文件到硬盘的过程也在以单独NSInvocationOperation完成，避免阻塞主线程。

这里借用一下在网上看到的一张非常详细的流程图：
![](https://ws3.sinaimg.cn/large/006tKfTcgy1g063b1fc81j321e0lrdms.jpg)

# 详细分析
## 相关工具类分析
### NSData + ImageContentType
&emsp;&emsp;这个分类提供了一个类方法`sd_contentTypeForImageData:`。通过这个方法传入图片的NSData数据，然后返回图片类型。图片类型通过SDImageFormat的枚举来定义。
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
    // 根据字母的ASCII码比较，返回对应的图片枚举类型
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

### SDWebImageCodersManager
&emsp;&emsp;编码解码管理器，处理多个图片编码解码任务，编码器是一个优先队列，这意味着后面添加的编码器将具有最高优先级。
```
@interface SDWebImageCodersManager : NSObject<SDWebImageCoder>

/// 管理器管理
+ (nonnull instancetype)sharedInstance;

/// 编码管理器中的所有编码器。编码器数组是一个优先级队列，这意味着后面添加的编码器将具有最高优先级
@property (nonatomic, strong, readwrite, nullable) NSArray<SDWebImageCoder>* coders;

/// 在编码器数组的末尾添加一个新的编码器，并且该编码器拥有最高的优先级
- (void)addCoder:(nonnull id<SDWebImageCoder>)coder;

/// 从编码器数组中删除一个编码器
- (void)removeCoder:(nonnull id<SDWebImageCoder>)coder;

@end
```
相关方法如下：
```
/// 添加编码器
- (void)addCoder:(nonnull id<SDWebImageCoder>)coder {
    /// 判断该编码器是否实现了SDWebImageCode协议
    if ([coder conformsToProtocol:@protocol(SDWebImageCoder)]) {
        dispatch_barrier_sync(self.mutableCodersAccessQueue, ^{
            [self.mutableCoders addObject:coder];
        });
    }
}

/// 删除编码器
- (void)removeCoder:(nonnull id<SDWebImageCoder>)coder {
    /// 同步删除编码器
    dispatch_barrier_sync(self.mutableCodersAccessQueue, ^{
        [self.mutableCoders removeObject:coder];
    });
}

/// 判断该图片是否可用编码
- (BOOL)canEncodeToFormat:(SDImageFormat)format {
    for (id<SDWebImageCoder> coder in self.coders) {
        if ([coder canEncodeToFormat:format]) {
            return YES;
        }
    }
    return NO;
}

/// 图像解码方法
- (UIImage *)decodedImageWithData:(NSData *)data {
    if (!data) {
        return nil;
    }
    for (id<SDWebImageCoder> coder in self.coders) {
        if ([coder canDecodeFromData:data]) {
            return [coder decodedImageWithData:data];
        }
    }
    return nil;
}

/// 压缩图像
- (UIImage *)decompressedImageWithImage:(UIImage *)image
                                   data:(NSData *__autoreleasing  _Nullable *)data
                                options:(nullable NSDictionary<NSString*, NSObject*>*)optionsDict {
    if (!image) {
        return nil;
    }
    for (id<SDWebImageCoder> coder in self.coders) {
        if ([coder canDecodeFromData:*data]) {
            return [coder decompressedImageWithImage:image data:data options:optionsDict];
        }
    }
    return nil;
}

/// 根据image和format（类型）编码图像
- (NSData *)encodedDataWithImage:(UIImage *)image format:(SDImageFormat)format {
    if (!image) {
        return nil;
    }
    for (id<SDWebImageCoder> coder in self.coders) {
        if ([coder canEncodeToFormat:format]) {
            return [coder encodedDataWithImage:image format:format];
        }
    }
    return nil;
}
```

### SDWebImageImageIOCoder
&emsp;&emsp;通过该类可以实现图片的解压缩操作，对于太大的图片，先按照一定比例缩小图片，然后再进行解压缩。
```
#if SD_UIKIT || SD_WATCH
// 每个像素占用的字节数
static const size_t kBytesPerPixel = 4;
// 色彩空间占用的字节数
static const size_t kBitsPerComponent = 8;
// 定义一张图片可以占用的最大空间
static const CGFloat kDestImageSizeMB = 60.0f;

static const CGFloat kSourceImageTileSizeMB = 20.0f;

static const CGFloat kBytesPerMB = 1024.0f * 1024.0f;
// 1MB可以存储多少像素
static const CGFloat kPixelsPerMB = kBytesPerMB / kBytesPerPixel;
// 如果像素小于这个值，则不解压缩
static const CGFloat kDestTotalPixels = kDestImageSizeMB * kPixelsPerMB;
static const CGFloat kTileTotalPixels = kSourceImageTileSizeMB * kPixelsPerMB;

static const CGFloat kDestSeemOverlap = 2.0f;
#endif

/// 判断是否可以解码
- (BOOL)canDecodeFromData:(nullable NSData *)data {
    switch ([NSData sd_imageFormatForImageData:data]) {
        case SDImageFormatWebP:
            // WebP类型不支持解码
            return NO;
        case SDImageFormatHEIC:
            // 检查HEIC解码的兼容性
            return [[self class] canDecodeFromHEICFormat];
        default:
            return YES;
    }
}

/// 判断是否可以解码，并且实现逐渐显示效果
- (BOOL)canIncrementallyDecodeFromData:(NSData *)data {
    switch ([NSData sd_imageFormatForImageData:data]) {
        case SDImageFormatWebP:
            // 不支持WebP渐进解码
            return NO;
        case SDImageFormatHEIC:
            // Check HEIC decoding compatibility
            return [[self class] canDecodeFromHEICFormat];
        default:
            return YES;
    }
}

/// 解压缩图片
- (nullable UIImage *)sd_decompressedImageWithImage:(nullable UIImage *)image {
    // 判断图片是否能解压缩
    if (![[self class] shouldDecodeImage:image]) {
        return image;
    }
    
    // autorelease the bitmap context and all vars to help system to free memory when there are memory warning.
    // on iOS7, do not forget to call [[SDImageCache sharedImageCache] clearMemory];
    // 解压缩操作放入一个自动释放池里面，以便自动释放所有的变量
    @autoreleasepool{
        
        CGImageRef imageRef = image.CGImage;
        // 获取图片的色彩空间
        CGColorSpaceRef colorspaceRef = [[self class] colorSpaceForImageRef:imageRef];
        
        size_t width = CGImageGetWidth(imageRef);
        size_t height = CGImageGetHeight(imageRef);
        
        // kCGImageAlphaNone is not supported in CGBitmapContextCreate.
        // Since the original image here has no alpha info, use kCGImageAlphaNoneSkipLast
        // to create bitmap graphics contexts without alpha info.
        CGContextRef context = CGBitmapContextCreate(NULL,
                                                     width,
                                                     height,
                                                     kBitsPerComponent,
                                                     0,
                                                     colorspaceRef,
                                                     kCGBitmapByteOrderDefault|kCGImageAlphaNoneSkipLast);
        if (context == NULL) {
            return image;
        }
        
        // Draw the image into the context and retrieve the new bitmap image without alpha
        // 绘制一个和图片大小一样的图片
        CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef);
        // 创建一个没有alpha通道的图片
        CGImageRef imageRefWithoutAlpha = CGBitmapContextCreateImage(context);
        // 得到解压缩以后的图片
        UIImage *imageWithoutAlpha = [[UIImage alloc] initWithCGImage:imageRefWithoutAlpha scale:image.scale orientation:image.imageOrientation];
        CGContextRelease(context);
        CGImageRelease(imageRefWithoutAlpha);
        
        return imageWithoutAlpha;
    }
}

// 是否需要减少原始图片的大小
+ (BOOL)shouldScaleDownImage:(nonnull UIImage *)image {
    BOOL shouldScaleDown = YES;
    
    CGImageRef sourceImageRef = image.CGImage;
    CGSize sourceResolution = CGSizeZero;
    sourceResolution.width = CGImageGetWidth(sourceImageRef);
    sourceResolution.height = CGImageGetHeight(sourceImageRef);
    // 图片总共像素
    float sourceTotalPixels = sourceResolution.width * sourceResolution.height;
    // 如果图片的总像素大于一定比例，则需要做简化处理
    float imageScale = kDestTotalPixels / sourceTotalPixels;
    if (imageScale < 1) {
        shouldScaleDown = YES;
    } else {
        shouldScaleDown = NO;
    }
    
    return shouldScaleDown;
}

// 获取图片的色彩空间
+ (CGColorSpaceRef)colorSpaceForImageRef:(CGImageRef)imageRef {
    // current
    CGColorSpaceModel imageColorSpaceModel = CGColorSpaceGetModel(CGImageGetColorSpace(imageRef));
    CGColorSpaceRef colorspaceRef = CGImageGetColorSpace(imageRef);
    
    BOOL unsupportedColorSpace = (imageColorSpaceModel == kCGColorSpaceModelUnknown ||
                                  imageColorSpaceModel == kCGColorSpaceModelMonochrome ||
                                  imageColorSpaceModel == kCGColorSpaceModelCMYK ||
                                  imageColorSpaceModel == kCGColorSpaceModelIndexed);
    if (unsupportedColorSpace) {
        colorspaceRef = SDCGColorSpaceGetDeviceRGB();
    }
    return colorspaceRef;
}
```

### SDWebImageCompat
&emsp;&emsp;该类就提供了一个全局方法`SDScaledImageForKey`，这个方法根据原始图片绘制一张放大或者缩小的图片。
```
/// 给定一张图片，通过scale属性返回一个放大的图片
inline UIImage *SDScaledImageForKey(NSString * _Nullable key, UIImage * _Nullable image) {
    if (!image) {
        return nil;
    }
    
#if SD_MAC
    return image;
#elif SD_UIKIT || SD_WATCH
    // 如果是动图，则迭代处理
    if ((image.images).count > 0) {
        NSMutableArray<UIImage *> *scaledImages = [NSMutableArray array];

        for (UIImage *tempImage in image.images) {
            [scaledImages addObject:SDScaledImageForKey(key, tempImage)];
        }
        // 把处理结束的图片再合成一张动态图片
        UIImage *animatedImage = [UIImage animatedImageWithImages:scaledImages duration:image.duration];
        if (animatedImage) {
            animatedImage.sd_imageLoopCount = image.sd_imageLoopCount;
        }
        return animatedImage;
    } else {        // 非动图的情况
#if SD_WATCH
        if ([[WKInterfaceDevice currentDevice] respondsToSelector:@selector(screenScale)]) {
#elif SD_UIKIT
        if ([[UIScreen mainScreen] respondsToSelector:@selector(scale)]) {
#endif
            CGFloat scale = 1;
            if (key.length >= 8) {
                NSRange range = [key rangeOfString:@"@2x."];
                if (range.location != NSNotFound) {
                    scale = 2.0;
                }
                
                range = [key rangeOfString:@"@3x."];
                if (range.location != NSNotFound) {
                    scale = 3.0;
                }
            }
           
            // 返回对应分辨率下的图片
            UIImage *scaledImage = [[UIImage alloc] initWithCGImage:image.CGImage scale:scale orientation:image.imageOrientation];
            image = scaledImage;
        }
        return image;
    }
#endif
}
```

# 总结
&emsp;&emsp;本文中，主要介绍了SDWebImage的基本架构、各种类的作用、相关工具类（NSData+ImageContentType、SDWebImageCodersManager、SDWebImageImageIOCoder以及SDWebImageCompat）的详细分析。在下文中[SDWebImage源码解析（二）](http://cloverkim.com/SDWebImage-source-code-analysis-2.html)，将会继续分析SDWebImage的相关核心类。