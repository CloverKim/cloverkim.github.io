---
title: iOS 使用Jenkins持续集成
categories: 开发工具
tags:
- 实践
- 持续集成
- Jenkins
top: 100
copyright: ture
---

# Jenkins 概述
## Jenkins是什么？
&emsp;&emsp;[Jenkins](https://jenkins.io/zh/) 是一款流行的开源持续集成（Continuous Integration，简称CI）工具，广泛用于项目开发，具有自动化构建、测试和部署等功能。通常与版本管理工具(SCM)、构建工具结合使用。常用的版本控制工具有SVN、GIT，构建工具有Maven、Ant、Gradle，当然也可以通过插件的方式安装Xcode构建工具。
&emsp;&emsp;CI是一种开发实践。实践应该包含3个基本模块，一个可以自动构建的过程，自动编译代码，可以自动分发，部署和测试。一个代码仓库，SVN或者Git。最后一个是一个持续集成的服务器。通过持续集成，可以让我们通过自动化等手段高频率地去获取产品反馈并响应反馈的过程。<!-- more -->

![](http://pic.cloverkim.com/Jenkins思维导图.png 'Jenkins思维导图')

## 优点
- 缩减开发周期，快速迭代版本
&emsp;&emsp;每个版本开始都会估算好开发周期，但是总会因为各种事情而延期。这其中包括了一些客观因素。由于产品线增多，迭代速度越来越快，给测试带来的压力也越来越大。如果测试都在开发完全开发完成之后再来测试，那就会影响很长一段时间。这时候由于集成晚就会严重拖慢项目节奏。如果能尽早的持续集成，尽快进入上图的12步骤的迭代环中，就可以尽早的暴露出问题，提早解决，尽量在规定时间内完成任务。

- 自动化流水线操作带来的高效
&emsp;&emsp;其实打包对于开发人员来说是一件很耗时，而且没有很大技术含量的工作。如果开发人员一多，相互改的代码冲突的几率就越大，加上没有产线管理机制，代码仓库的代码质量很难保证。团队里面会花一些时间来解决冲突，解决完了冲突还需要自己手动打包。这个时候如果证书又不对，又要耽误好长时间。这些时间其实可以用持续集成来节约起来的。一天两天看着不多，但是按照年的单位来计算，可以节约很多时间！

- 随时可部署
&emsp;&emsp;有了持续集成以后，我们可以以天为单位来打包，这种高频率的集成带来的最大的优点就是可以随时部署上线。这样就不会导致快要上线，到处是漏洞，到处是bug，手忙脚乱弄完以后还不能部署，严重影响上线时间。

- 极大程度避免低级错误

# Jenkins 安装
&emsp;&emsp;在安装Jenkins之前，要保证Mac已经安装好了Java环境，因为Jenkins的运行需要依赖于Java环境。
## pkg镜像安装
&emsp;&emsp;pkg镜像包[下载地址](https://jenkins.io/zh/download/)，下载完安装，直接安装，一路下一步即可，比较简单。

## Homebrew
&emsp;&emsp;如果有不了解Homebrew的同学，可以先了解一下（[Homebrew官网](https://brew.sh/index_zh-cn)）。
```
// 安装命令
$ brew install jenkins
/// 启动Jenkins
$ jenkins
```

## war包方式安装（推荐）
&emsp;&emsp;推荐使用war包的方式进行安装，可以指定端口号，方便端口号的管理使用。[war包下载地址](http://updates.jenkins-ci.org/download/war/)
```
使用命令
$ java -jar /Documents/jenkins.war –httpPort=8080
```

# Jenkins配置
## 初始化
- Unlock Jenkins
&emsp;&emsp;首次启动Jenkins，会要求对Jenkins进行解锁操作，在下图中显示的路径`Users/Shared/Jenkins/Home/secrets/initialAdminPassword`中存放着解锁需要的密码。我们可以直接在终端中使用`cat`命令就可获取到密码，直接粘贴，continue。
    ![](http://pic.cloverkim.com/unlockjenkins.png 'Unlock Jenkins')
    
- Customize Jenkins
&emsp;&emsp;接下来是对Jenkins进行定制，有安装推荐的插件以及选择插件进行安装，这里选择安装的推荐插件即可。
![](http://pic.cloverkim.com/customize-jenkins.png 'Customize Jenkins')
![](http://pic.cloverkim.com/plug-install.png '插件安装安装')

- Create First Admin User
&emsp;&emsp;最后一步为，创建一个管理员用户。根据自己的信息新建一个用户即可。
![](http://pic.cloverkim.com/create-admin.png '新建Admin用户')

## 插件安装
### 常用插件介绍
- Xcode integration（该插件为构建者提供了构建xcode项目，调用agvtool和打包.ipa文件的工具）
- GIT plugin
- GitLab Plugin
- GitLab Hook Plugin
- Keychains and Provisioning Profiles Management（证书管理插件）
- Branch API Plugin（多分支管理插件）
- Build Name and Description Setter（构建名称和构建完成描述设置插件）
- Build With Parameters（多参数构建设置）
- description setter plugin（构建完成日志描述）
- Dingding[钉钉] Plugin（钉钉通知推送插件）
- fir-plugin（上传到fir.im必备插件）
- Localization: Chinese (Simplified)（中文本地化插件）
- Project Description Setter
- Role-based Authorization Strategy（用户权限管理）
- Version Number Plug-In（允许构件时修改项目版本号）

### 安装
- 搜索安装
&emsp;&emsp;从Manage Jenkins - Manage Plugins，进入插件管理页面，在可选插件tab以及右上角的过滤中，搜索需要安装的插件，勾选对应的插件，然后直接点击下方的安装，等待安装完毕以及Jenkins重启完毕即可。
![](http://pic.cloverkim.com/20200116155027.png '插件安装')

- 本地上传
&emsp;&emsp;依然是在插件管理页面， 在高级tab页中，在页面下方的上传插件进行上传下载的.hpi插件文件。剩下步骤同搜索安装。
![](http://pic.cloverkim.com/20200116155447.png '本地上传')

&emsp;&emsp;以下相关教程，均在以上插件已安装完成的条件下进行。

## keychain和证书上传配置
&emsp;&emsp;要出现管理证书入口，要先安装证书管理插件（Keychains and Provisioning Profiles Management），然后从Manage Jenkins - Keychains and Provisioning Profiles Management入口中，进入证书管理页面，先配置Keychains。对于login.keychain文件，在/Users/管理员用户名/Library/keychains/ 目录下，但是由于系统的关系，当前的keychains名称为login.keychain-db，这里我们需要复制一份到另外一个目录下，将其修改为login.keychain即可继续上传。证书的代码签名身份，在“钥匙串”App的“我的证书”里面能获取到。
![](http://pic.cloverkim.com/20200116162055.png '添加keychains')
&emsp;&emsp;接下来上传所需要的证书。在/Users/管理员用户名/Library/MobileDevice/Provisioning Profiles目录下，找到打包需要的证书，这里我只用到了ADHoc的证书，并将其复制到一个指定的目录中，这里我复制到了桌面的MobileDevice目录下。证书的上传同上一步keychain的上传。
![](http://pic.cloverkim.com/20200116162936.png '上传证书')

&emsp;&emsp;如果是从开发者官网下载的证书，这里分享一个用终端的方式，获取证书对应的UUID，[命令行获取mobileprovision文件的UUID](https://my.oschina.net/ioslighter/blog/494342)
![](http://pic.cloverkim.com/20200116164456.png '获取证书UUID')

# 创建项目、配置
## 创建
&emsp;&emsp;从首页的左侧菜单新建item中，进入项目新建页面，创建步骤，如下图所示。
![](http://pic.cloverkim.com/20200116191327.png '创建步骤')

## General配置
![](http://pic.cloverkim.com/20200116194838.png 'General')
&emsp;&emsp;如上图所示，下列我们针对常用3项进行详细说明，分别为项目描述、抛弃旧的构建任务以及多参数构建配置。
1. 描述 - 填写项目的具体描述

2. 抛弃旧的构建任务
    &emsp;&emsp;分别有以下丢弃策略，这里我们一般用前面2种策略，保持构建的天数或者构建的最大个数。
    - 保持构建的天数
    - 保持构建的最大个数
    - 发布包保留天数
    - 发布包最大保留#个构建

3. This project is parameterized（多参数构建配置）
&emsp;&emsp;针对多参数构建，这里只对以下常用的配置进行说明，配置完成的结果如下图所示：
    - Boolean Parameter（布尔值参数类型，比如用于是否为Jenkins打包，一些参数控制等）
    - Choice Parameter（选择参数，比如用户打包环境的选择）
    - Git Parameter（Git参数，用于Branch/Tag/Revision/Pull Request选择构建）
    - String Parameter（字符串类型参数，比如用于版本号以及build版本号的使用等）
![](http://pic.cloverkim.com/20200116201036.png '多参数配置示例')
![](http://pic.cloverkim.com/20200117094515.png '使用示例')

## 源代码管理
&emsp;&emsp;因为基本是是用Git来管理代码，因此这里只介绍Git的方式。一般公司会搭建属于自己的gitlab仓库，这里的仓库URL就要到自己公司的代码仓库进行查看，凭据的用户名和密码就是自己登陆gitlab的用户名和密码。这里特别说明下，指定分支，不要用默认以及写死，这里要用到上面配置的多参数管理中的Git Parameter，用`$`来引用定义的name，比如`$BRANCH`，在构建时，选择对应的分支就会拉取对应的分支代码。
![](http://pic.cloverkim.com/20200117100456.png '源代码管理')

## 构建触发器
&emsp;&emsp;构建触发器主要是设置自动化测试的配置，触发器可自定义的地方很多，可以根据自己实际的项目进行配置，这里不做多说明和研究了。主要讲下下面的定时构建和轮询SCM。
- 定时构建：不管是SVN或者Git中代码是否有提交，均执行定时化的构建任务。
- 轮询SCM：只要SVN或者Git中代码有更新，则执行构建任务，日程表的填写内容有个参数，从左到右的参数含义如下：
    - 分钟minute，取值0~59
    - 小时hour，取值0~23
    - 天day，取值1~31
    - 月month，取值1~12
    - 星期week，取值0~7（0和7都是表示星期日）

&emsp;&emsp;以上5个参数可选择性设定，不写死的参数用`*`代替参数之间用空格隔开，例如：
```
"0 21 * * *"表示每晚21点0分自动化构建一次
"0 * * * *"表示每个小时的第0分钟执行一次构建
"H/5 * * * *"每隔5分钟构建一次
"H H/2 * * *"每两小时构建一次
"H H 30 * *"每月30号构建一次
"H(0-29)/10 * * * *"每个小时的前半个小时内的每10分钟
"0 8-17/2 * * 1-5"周一到周五，8点~17点，两小时构建一次
"H H 1,15 1-11 *"每月1号、15号各构建一次，除12月等
```
![](http://pic.cloverkim.com/20200117103025.png '构建触发器')

## 构建环境
&emsp;&emsp;在构建环境中，有很多配置项。但这里只针对以下几项常用项进行说明`Keychains and Code Signing Identities`、`Mobile Provisioning Profiles`、`Set jenkins user build variables`。
- Keychains and Code Signing Identities：keychains和签名身份，这里用的是之前在证书管理页面中进行配置的，这我们选择项目所对应的keychains和签名身份即可，示例是下图所示：
![](http://pic.cloverkim.com/20200117105500.png)

- Mobile Provisioning Profiles：证书文件，这里用到的也是在证书管理页面上传配置的，这里只需要选择对应的证书名称，其余项不需要手动配置，示例如下图所示：
![](http://pic.cloverkim.com/20200117105704.png)

- Set jenkins user build variables：设置jenkins用户构建变量，这个选项是需要安装“user build vars plugin”插件的，该选项的说明如下：
![](http://pic.cloverkim.com/20200117110002.png 'Set jenkins user build variables')

## 构建步骤
### 使用脚本构建
#### 自动化打包命令xcodebuild + xcrun介绍
&emsp;&emsp;Xcode为开发者提供了一套构建打包的命令，为xcodebuild何xcrun命令。xcodebuild把指定的项目打包成.app文件，xcrun将制定的.app文件转换成对应的.ipa文件。
- xcodebuild - 相关文档如下：

```
NAME
xcodebuild – build Xcode projects and workspaces

SYNOPSIS
1. xcodebuild [-project name.xcodeproj] [[-target targetname] … | -alltargets] [-configuration configurationname] [-sdk [sdkfullpath | sdkname]] [action …] [buildsetting=value …] [-userdefault=value …]

2. xcodebuild [-project name.xcodeproj] -scheme schemename [[-destination destinationspecifier] …] [-destination-timeout value] [-configuration configurationname] [-sdk [sdkfullpath | sdkname]] [action …] [buildsetting=value …] [-userdefault=value …]

3. xcodebuild -workspace name.xcworkspace -scheme schemename [[-destination destinationspecifier] …] [-destination-timeout value] [-configuration configurationname] [-sdk [sdkfullpath | sdkname]] [action …] [buildsetting=value …] [-userdefault=value …]

4. xcodebuild -version [-sdk [sdkfullpath | sdkname]] [infoitem]

5. xcodebuild -showsdks

6. xcodebuild -showBuildSettings [-project name.xcodeproj | [-workspace name.xcworkspace -scheme schemename]]

7. xcodebuild -list [-project name.xcodeproj | -workspace name.xcworkspace]

8. xcodebuild -exportArchive -archivePath xcarchivepath -exportPath destinationpath -exportOptionsPlist path

9. xcodebuild -exportLocalizations -project name.xcodeproj -localizationPath path [[-exportLanguage language] …]

10. xcodebuild -importLocalizations -project name.xcodeproj -localizationPath path
```
上面所罗列的10个命令中，最主要的是前3个，接下来说明一下参数：
1. `-project -workspace`：这两个对应的为项目的名字，如果有多个工程，在没有指定的情况下，则默认为第一个工程。
2. `-target`：打包对应的targets，如果没有指定，则默认为第一个。
3. `-configuration`：如果没有修改这个配置，默认为Debug和Release两种方式，没有指定则默认为Release方式。
4. `-buildsetting=value ...`：使用此命令修改工程的配置。
5. `-scheme`：指定打包的scheme。

- xcrun
&emsp;&emsp;使用参数不多，使用方法也很简单，xcrun -sdk iphoneos -v PackageApplication + 上述一些参数。

```
Usage:
PackageApplication [-s signature] application [-o output_directory] [-verbose] [-plugin plugin] || -man || -help

Options:

[-s signature]: certificate name to resign application before packaging
[-o output_directory]: specify output filename
[-plugin plugin]: specify an optional plugin
-help: brief help message
-man: full documentation
-v[erbose]: provide details during operation
```

- 完整的构建脚本示例

```
## !/bin/sh
## 项目名
TARGET_NAME=Your target name
## Scheme名
SCHEME=Your scheme name
##=======================
## 编译类型
BUILD_TYPE=${BUILD_TYPE}
## 当前目录
SORCEPATH=${WORKSPACE}
## workspace名
SPACE=${WORKSPACE}/${TARGET_NAME}.xcodeproj
##xcarchive文件的存放路径
ARCHIVEPATH=$SORCEPATH/build/$SCHEME.xcarchive
## ipa文件的存放路径
EXPORTPATH=$SORCEPATH/build/$SCHEME
## ExportOptions.plist文件的存放路径，该文件要存放在这个路径下内容如下
EXPORTOPTIONSPLIST=$SORCEPATH/build/ExportOptions.plist
## 导出后的ipa路径
EXPORTPATHIPA=$SORCEPATH/build/$SCHEME/$SCHEME.ipa

echo -e "============First Build Clean============"
## 清理缓存
## 如果工程使用的是cocoapods，则'-project %s.xcodeproj'替换为'-workspace %s.xcworkspace'
xcodebuild clean -project $SPACE -scheme ${SCHEME} -configuration ${BUILD_TYPE}
echo -e "============Build Clean============"
## 输出关键信息
echo -e "  TARGET_NAME    : ${TARGET_NAME}"
echo -e "  BUILD_TYPE    : ${BUILD_TYPE}"
echo -e "  SORCEPATH    : ${SORCEPATH}"
echo -e "  ARCHIVEPATH    : ${ARCHIVEPATH}"
echo -e "  EXPORTPATH    : ${EXPORTPATH}"
echo -e "  EXPORTOPTIONSPLIST    : ${EXPORTOPTIONSPLIST}"
echo -e "============Build Archive============"

## 导出archive包
xcodebuild archive -project ${SPACE} -scheme ${SCHEME} -archivePath $ARCHIVEPATH
echo -e "============Build Archive Success============"

echo -e "============Export IPA============"
## 导出IPA包
xcodebuild -exportArchive -archivePath $ARCHIVEPATH -exportPath ${EXPORTPATH} -exportOptionsPlist ${EXPORTOPTIONSPLIST}
echo -e "============Export IPA SUCCESS============"

## 编译完成时间 20181030_0931
BUILD_DATE="$(date +'%Y%m%d_%H%M')"

## info.plist路径
PROJECT_INFOPLIST_PATH="${SORCEPATH}/${TARGET_NAME}/Info.plist"
## 取版本号
BUNDLESHORTVERSION=$(/usr/libexec/PlistBuddy -c "print CFBundleShortVersionString" "${PROJECT_INFOPLIST_PATH}")
## 取build值
VERSION=$(/usr/libexec/PlistBuddy -c "print CFBundleVersion" "${PROJECT_INFOPLIST_PATH}")
## ipa更名规则  项目名V版本_年月日_时分
IPANAME="${TARGET_NAME}V${BUNDLESHORTVERSION}_${BUILD_DATE}.ipa"
## 更名后ipa路径
EXPORTPATHNEWIPA=$EXPORTPATH/$IPANAME

echo -e "============Export end :${BUILD_DATE}============"
echo -e "============IPA Old Name: ${EXPORTPATHIPA}============"
echo -e "============IPA New Name: ${EXPORTPATHNEWIPA}============"

## IPA更名
cp $EXPORTPATHIPA $EXPORTPATHNEWIPA
echo -e "============Create New Name Success============"
## 删除老IPA
rm $EXPORTPATHIPA
echo -e "============Delete Old Name Success============"
```

### 使用xcode插件构建
#### General build settings（常规配置）
常规的配置见下图所示，其他选项保持默认即可。
![](http://pic.cloverkim.com/20200117175423.png)
![](http://pic.cloverkim.com/20200117180315.png)

#### Code signing & OS X keychain options（keychain和签名证书管理）
![](http://pic.cloverkim.com/20200117181507.png)

#### Advanced Xcode build options（构建高级选项）
&emsp;&emsp;构建的高级选项，主要设置支持cocoaPods的项目，需要配置workspace和项目目录以及项目文件，示例如下图所示：
![](http://pic.cloverkim.com/20200117111050.png '构建高级选项')

#### Versioning（构建版本管理）
&emsp;&emsp;这里的版本控制，主要是为了在打包构建的时候，能手动输入版本号进行控制，从而更好的输出不同版本号的包进行测试，如下图所示，其中VERSION_NAME和BUILD_VERSION是在General中配置的多参数打包的String 参数，名称要一一对应。
![](http://pic.cloverkim.com/20200117110536.png '版本控制')

## 构建后的操作
- 输出构建完成信息 - Archive artifacts
&emsp;&emsp;增加一个构建后的操作步骤：Archive artifacts，`build/${BUILD_TYPE}-iphoneos/*.ipa`，这里用到的目录需要跟“General build settings”步骤中的Output directory输出目录进行对应，不然会匹配不到相对应的内容，如果需要输出所有构建结果，不用后面的通配符即可。如下图所示，虽然有报错，但是根据错误内容，是没任何问题的，保存即可。
![](http://pic.cloverkim.com/20200117112049.png 'Archive artifacts')

- 上传到fir.im/蒲公英
    - fir.im，项目的配置如下图所示：
    ![](http://pic.cloverkim.com/20200117114521.png 'fir.im配置')

    - 蒲公英：因为公司用到的是fir.im，所以暂时没有蒲公英的账号以及配置过程，详细的可以参考蒲公英官网的配置文档：[使用 Jenkins 插件上传应用到蒲公英](https://www.pgyer.com/doc/view/jenkins_plugin)

- 钉钉消息通知
&emsp;&emsp;配置钉钉消息通知，这里需要安装`Dingding[钉钉] Plugin`和`Dingding JSON Pusher Plugin`插件。示例如下图所示：
![](http://pic.cloverkim.com/20200117113608.png '钉钉消息通知')
&emsp;&emsp;关于钉钉access token的字段，这里分享[如何创建一个钉钉机器人](https://ding-doc.dingtalk.com/doc#/serverapi2/qf2nxq)，添加完成后，将Webhook链接里的额access_token复制到Jenkins里即可。

# 用户权限管理
&emsp;&emsp;Jenkins默认的权限管理并没有用户组的概念，但在实际工作中经常需要针对不同的用户组赋予不同的权限，通常需要第三方插件（`Role-based Authorization Strategy`）的支持来解决这个问题。
## 设置授权策略
&emsp;&emsp;安装完`Role-based Authorization Strategy`插件之后，进入`系统管理 - Configure Global Security`，授权策略会多出一个`Role-Based Strategy`选项，选择此项，其他设置如下图所示：
![](http://pic.cloverkim.com/20200117193736.png 'Configure Global Security')

## 创建用户
&emsp;&esmp;从`系统管理 - 管理用户 - 新建用户`中，创建用户，这里新建3个测试用户：
![](http://pic.cloverkim.com/20200117194347.png '新建用户')

## 管理和分配角色
&emsp;&emsp;当启用了角色授权策略后，默认只有admin用户拥有系统配置和项目的所有权限，其他用户需要添加角色和权限才可以，不然会没有任何权限。从`系统管理 - Manage and Assign Roles`进入管理和分配角色页面。

### Manager Roles 管理角色
&emsp;&emsp;对角色进行增加删除、权限设置，这里可以创建全局角色和项目角色，并分配不同的权限。
#### 全局角色、项目角色
- 全局角色：默认情况下，全局角色中只有一个admin角色，拥有所有权限。通过`Role to add`添加自定义的全局角色，并通过矩阵选择的方式添加权限。这里添加两个全局角色，如下图所示，其中，global_test角色拥有读取Jenkins配置的权限、项目的增删改查等所有权限。others角色只有读取Jenkins配置的权限。
![](http://pic.cloverkim.com/20200117195942.png '新增全局角色')

- 项目角色：配置项目角色的步骤跟全局角色类似，但是要设置pattern，用于匹配指定类型的项目，支持正则表达式。在这里，创建了如图所示的两个项目角色。
其中，web角色使用`web.*`匹配以web开头的项目，sample角色使用`sample.*`匹配以sample开头的项目。
![](http://pic.cloverkim.com/20200117200215.png '新增项目角色')

#### 两者的区别
- 两者都需要通过`Role to add`添加角色。
- 两者的角色都是通过勾选对应的全选矩阵来设置不同的角色权限。
- 全局角色既可以设置管理Jenkins的配置，也可以设置管理项目，项目角色只能设置管理项目，没有管理Jenkins的权限配置。
- 添加项目角色时，要通过pattern设置角色可以匹配的项目名称，pattern支持正则表达式。
    - 如`Roger-.*`表示所有以`Roger-`开头的项目，区分大小写。
    - `(?i)roger-.*`表示以`roger-`开头的项目，不区分大小写。
    - `ABC|ABC.*`表示`ABC`或者以`ABC`开头的项目。
    - `abc|bcd|efg`表示直接匹配多个项目。

#### 各种权限说明
&emsp;&emsp;权限说明如下表所示，这里对部分不常用的权限进行了缩减。

| Overall |  | Credentials（凭据） |  |  |  |  | Job（任务） |  |  |  |  |  |  |  |  | View（视图） |  |  |  |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| Administer | Read | Create | Delete | ManageDomains | Update | View | Build | Cancel | Configure | Create | Delete | Discover | Move | Read | Workspace | Configure | Create | Delete | Read |
| 管理员 | 可读 | 新增 | 删除 | 管理域 | 更新 | 查看 | 构建 | 取消 | 配置 | 创建 | 删除 | 重定向 | 移动 | 可读 | 查看工作区 | 配置 | 创建 | 删除 | 可读 |

&emsp;&emsp;其中有一些比较特别的权限：
- 最大的权限是Overall的Administer，拥有该权限可以对Jenkins进行任何操作。
- 最基本的权限是Overall的Read，用户必须赋予可读的权限。
- 给用户赋予权限时，Overall的Read和Job的Read权限要同时选中，不然该用户无权限访问该Job。
- 其他都是一些基本的权限，根据实际情况进行选择。

### Assign Roles 分配角色（对已经创建的角色分配具体的用户）
&emsp;&emsp;回到`Manage and Assign Roles`，进入`Assign roles`，在这里给全局角色和项目角色分配用户。如下图所示：这里分别对test1用户设置了admin的全局角色权限，对test2用户设置了global_test的全局角色权限，对test3用户设置了sample的项目角色权限。现在角色和用户已经配置完毕，可以使用不同的用户进行登录，查看不同的设置结果。
&emsp;&emsp;这里需要特别注意的地方，添加用户的时候，必须是之前已经创建好的用户。
![](http://pic.cloverkim.com/20200117203108.png '分配全局角色')
![](http://pic.cloverkim.com/20200117203128.png '分配项目角色')

# 完整的持续集成流程
&emsp;&emsp;经过上面的持续集成，完整持续集成的流程如下图所示：
![](http://pic.cloverkim.com/20200117203554.png '完整持续集成')

# 参考
- [简书 - 一缕殇流化隐半边冰霜 - 手把手教你利用Jenkins持续集成iOS项目](https://www.jianshu.com/p/41ecb06ae95f/)
- [掘金 - 72行代码 - iOS 使用Jenkins持续集成(简称CI)](https://juejin.im/post/5dde3ecde51d45771465b5aa?utm_source=gold_browser_extension)
- [CSDN - 媛测 - jenkins 用户权限管理](https://blog.csdn.net/lijing742180/article/details/86552396)