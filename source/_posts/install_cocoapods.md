---
title: CocoaPods安装教程
categories: 开发工具
tags:
- iOS
- CocoaPods
- Mac
top: 100
copyright: ture
---

# CocoaPods介绍
&emsp;&emsp;我们在iOS开发中不可避免的要使用第三方开源库，而CocoaPods的作用就是使我们方便管理应用中的第三方开源库。CocoaPods的项目源码在[https://github.com/CocoaPods/Specs](https://github.com/CocoaPods/Specs)上管理。
<!-- more -->
# 为什么要使用CocoaPods
&emsp;&emsp;在使用CocoaPods之前，我们需要把用到的第三方开源库的源代码复制到项目中，而这些开源库通常需要依赖系统的一些framework，我们需要手动的将这些framework逐一的添加到项目依赖中，同时我们也要管理这些依赖包的更新。这些操作虽然简单但毫无技术含量而且浪费时间。在使用CocoaPods之后，我们只需要把用到的第三方开源库放到一个名为podfile的文件中，然后执行pod install，CocoaPods会自动将这些第三方开源库的源码下载下来，并且为我们的项目配置好相应的系统依赖和编译参数。

# CocoaPods的安装
## 替换ruby源
CocoaPods是基于ruby ecosystem的，需要ruby环境，使用ruby的gem命令。如果有VPN，则无需替换原本的ruby源。默认的ruby源为：
``` 
https://rubygems.org/
```
接下来替换为墙内可以访问的ruby-china的镜像
```
// 1.移除掉原有的源
gem sources --remove https://rubygems.org/
// 2.淘宝的源已经不更新维护，现在使用ruby-china的源
gem source -a https://gems.ruby-china.com/
// 3.验证是否替换成功
gem sources -l
```

## 更新gem版本
```
sudo gem update --system
```
更新完毕后，可以通过以下命令查看gem版本
```
gem -v
```

## 安装CocoaPods
使用以下安装命令，安装最新版本的CocoaPods
```
sudo gem install -n /usr/local/bin cocoapods
```
安装成功如下图所示：
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fw37mrk9l9j20i005njsj.jpg)
安装成功之后，可以使用命令pod --version查看是否安装成功，如果成功会显示pod的版本号信息。

## 更新Podspec索引文件
pod安装成功之后，第一个操作就是执行以下命令：
```
pod setup
```
pod setup的作用：将所有第三方的Podspec索引文件更新到本地的~/.cocoapods/repos目录下
第一次执行pod setup命令时，github的Podspec索引文件比较大，所以第一次更新会比较慢，需要耐心等耐完成...
- 如果没有执行过pod setup，用户根目录下~找不到.cocoapods/repos目录，没有创建这个目录。
- 如果执行了pod setup，并且命令没有执行成功，那么会创建~/.cocoapods/repos目录，但该目录为空。
- 如果执行了pod setup，并且命令执行成功，说明已经把github上的Podspec文件更新到本地，那么会创建~/.cocoapods/repos目录，并且repos目录里会有一个master目录，这个master目录保存的就是github上所有第三方开源库的Podspec索引文件。
最后pod setup成功时，如下图所示：
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fw38vsef5sj20hw0afwgl.jpg)

# 参考
- [https://blog.csdn.net/jiankeufo/article/details/79362660](https://blog.csdn.net/jiankeufo/article/details/79362660)
- [https://blog.csdn.net/sinat_30898863/article/details/51348057](https://blog.csdn.net/sinat_30898863/article/details/51348057)