---
title: iOS开发 - 删除连连支付libRsaCrypto.a中的base64.o
categories: iOS开发
tags:
- iOS
- 第三方库
top: 100
copyright: ture
---
# 问题的开始
&emsp;&emsp;之前项目需要集成连连支付，根据连连支付文档集成之后，发现编译报了如图1-1 所示的错误，即libRsaCrypto.a中的base64.o和其他库冲突了，在网上进行了搜索与一番尝试，比如在linking—>other linker flags使用-force_load导入我们的libRsaCrypto.a，但还是无补于事。最后找到了一个比较简单粗暴的方法，既然libRsaCrypto.a中的base64.o和其他库冲突了，那就将其base64.o删除，前提是确定删除base64.o对原有的第三方库没有影响。
<!-- more -->
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fw2wubwxvij219g046768.jpg '1-1')

 附带介绍下－ObjC、－all_load、-force_load 3个参数的作用： 
- -ObjC：加了这个参数后，链接器就会把静态库中所有的Objective-C类和分类都加载到最后的可执行文件中。
- -all_load： 会让链接器把所有找到的目标文件都加载到可执行文件中，但是千万不要随便使用这个参数！假如你使用了不止一个静态库文件，然后又使用了这个参数，那么你很有可能会遇到ld: duplicate symbol错误，因为不同的库文件里面可能会有相同的目标文件，所以建议在遇到-ObjC失效的情况下使用-force_load参数。
- -force_load ：所做的事情跟-all_load其实是一样的，但是-force_load需要指定要进行全部加载的库文件的路径，这样的话，你就只是完全加载了一个库文件，不影响其余库文件的按需加载。

# 开始删除libRsaCrypto.a中的base64.o
## 1、查看libRsaCrypto.a文件支持的cpu架构
我们在终端中，先进入到.a文件所在的目录，然后输入相应的命令
``` 
lipo -info libRsaCrypto.a 
```
输出结果如图2-1 所示：
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fw2wuc64t4j20uc03cdhm.jpg '2-1')
## 2、根据上面的输出平台进行拆分.a文件
``` 
lipo libRsaCrypto.a -thin armv7 -output 1/libRsaCrypto-armv7.a
lipo libRsaCrypto.a -thin i386 -output 2/libRsaCrypto-i386.a 
lipo libRsaCrypto.a -thin x86_64 -output 3/libRsaCrypto-x86_64.a
lipo libRsaCrypto.a -thin arm64 -output 4/libRsaCrypto-arm64.a
```
这里需要注意的是：拆分后的.a文件在放在不同的文件夹下，这里我用当前目录下的文件夹1/2/3/4代替了。效果图如图2-2 所示：
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fw2wud4b2qj20my04ewer.jpg '2-2')
## 3、解压每个拆分后的.a文件
注意在解压前，先进入拆分后.a对应的文件夹下，在解压的命令分别在不同的文件夹下执行。
```
ar -x libRsaCrypto-armv7.a
ar -x libRsaCrypto-i386.a
ar -x libRsaCrypto-x86_64.a
ar -x libRsaCrypto-arm64.a
```
解压完后，对应的文件效果如图2-3 所示：
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fw2wuc0afcj20bm0esab1.jpg '2-3')
解压后可以删除对应的.o文件，这里我们需要删除的是base64.o。注意：要在每个文件夹里把对应的文件全部删除。然后再将解压之前的.a文件进行删除，即libRsaCrypto-armv7.a、libRsaCrypto-i386.a、libRsaCrypto-x86_64.a、libRsaCrypto-arm64.a。这样做，是为了方便我们重新命名合并后的.a文件。
## 4、合并解压后的.o文件
分别进入对应的文件夹，对所有解压出来的.o文件进行合并。命令如下：
```
libtool -static -o libRsaCrypto-armv7.a *.o
libtool -static -o libRsaCrypto-i386.a *.o
libtool -static -o libRsaCrypto-x86_64.a *.o
libtool -static -o libRsaCrypto-arm64.a *.o
```
经过一番合并命令后，对应的文件夹会重新生成对应的.a文件。
## 5、最后合并重新生成的.a文件
首先将每个文件夹下重新生成的.a文件拷贝到一个新的文件下。然后在终端键入合并命令：
```
lipo -create -output libRsaCrypto.a libRsaCrypto-armv7.a libRsaCrypto-i386.a libRsaCrypto-x86_64.a libRsaCrypto-arm64.a
```
到这一步，我们已经将libRsaCrypto.a里的base64.o进行了删除，并且重新合并生成了新的libRsaCrypto.a。
最后的最后，将重新合并生成的libRsaCrypto.a重新导入我们的项目中，尽情的 command + R 吧。
# 扩展
当我们集成其他第三方库也遇到类似的冲突问题时，也可以用此方法，不过需要确认第三方库的.a文件并不依赖我们将要删除的.o文件！！！


