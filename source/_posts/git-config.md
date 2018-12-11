---
title: Git使用前配置
categories: 开发工具
tags:
- Git
- Mac
- 实践
top: 100
copyright: ture
---

# 配置用户信息
&emsp;&emsp;在安装完Git时候，第一件事就是设置我们的用户名称和邮箱地址，因为每一次Git的提交都会使用这些信息，并且它会写入到每一次的提交当中，不可更改。如果是应用于多人协助开发的话，用户名和邮箱地址要按照公司部门的规范来进行设置。<!-- more -->
```
$ git config --global user.name "cloverkim"
$ git config --global user.email "jiang.shunjin@icloud.com"
```

# 配置SSH key
1、首先查看本地是否存在SSH-Key，打开终端，输入命令`ls ~/.ssh`，如果结果如下图所示，则说明本地已经存在SSH Key，则跳过创建的步骤，直接到第3步。
![](https://ws1.sinaimg.cn/large/749c46aagy1fy1tkaqhwmj206g02x74b.jpg)

2、如果本地没有生成过SSH Key，则需要生成，相关命令如下：
```
ssh-keygen -t rsa -C"you_email"
```
&emsp;&emsp;your_email：填写自己的邮箱地址，在终端输入命令后，根据提示直接敲回车即可，生成成功后如下图所示：
![](https://ws1.sinaimg.cn/large/749c46aagy1fy1txneeedj20fi095mya.jpg 'ssh-key')

3、然后在终端执行命令` cat ~/.ssh/id_rsa.pub `，将我们的ssh key进行复制，然后打开代码托管平台的设置界面，这里以[码云](https://gitee.com/)为例。点击安全设置的SSH公钥选项，根据情况添加标题、将刚刚复制的ssh key粘贴到③处，然后点击确定即可，这就在码云上配置好了我们的ssh key。
![](https://ws1.sinaimg.cn/large/749c46aagy1fy1u5qayc7j20uh0iqn0j.jpg '码云')