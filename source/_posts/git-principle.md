---
title: Git原理简介和常用命令
categories: 开发工具
tags:
- Git
- Mac
top: 100
copyright: ture
---

# 前言
&emsp;&emsp;在我们日常的开发工作中，对代码的管理一般会用到Git，当然也有使用SVN。对于刚接触Git或者仅仅是使用Git pull、Git push等几个常用命令的人来说，对Git还是挺陌生的，不理解其原理和如何具体使用。本文会对Git的原理以及常用命令进行介绍，希望对刚接触Git的开发者有一定的帮助。
<!-- more -->
# Git基础和原理
## 直接记录快照，而非差异比较
&emsp;&emsp;Git和其他版本控制系统（包括SVN和近似工具）的主要差别在于Git对待数据的方法。概念上来区分，其他大部分系统以文件变更列表的方式存储信息。这类系统（CVS、Subversion、Perforce、Bazaar等等）将它们保存的信息看作是一组基本文件和每个文件随时间逐步累积的差异。Git不按照以上方式对待或保存数据。反之，Git更像是把数据看作是对小型文件系统的一组快照。每次你提交更新，或在Git中保存项目状态时，它主要对当时的全部文件制作一个快照并保存这个快照的索引。为了高效，如果文件没有修改，Git不再重新存储该文件，而是只保留一个链接指向之前存储的文件。Git对待数据更像是一个快照流。如下图所示。这是Git与几乎所有其它版本控制系统的重要区别。
![](https://ws1.sinaimg.cn/large/749c46aagy1fxwutjhlfrj20m808hwgu.jpg)

## 近乎所有操作都是本地执行
&emsp;&emsp;在Git中的绝大多数操作只需要访问本地文件和资源，一般不需要来自网络上其它计算机的信息。如果你习惯于所有操作都有网络延时开销的集中式版本控制系统，Git在这方面会让你感到速度之神赐给了Git超凡的能量。因为你在本地磁盘上就有项目的完整历史，所以大部分操作看起来瞬间完成。
&emsp;&emsp;举个🌰，要浏览项目的历史，Git不需要外连到服务器去获取历史，然后再显示出来，它只需要直接从本地数据库中读取。你能立即看到项目历史。如果想查看当前版本与一个月前的版本之间引入的修改，Git会查找到一个月前的文件做一次本地的差异计算，而不是由远程服务器处理或从远程服务器拉回旧版本文件再来本地处理。

## Git 保证完整性
&emsp;&emsp;Git中所有数据在存储前都计算校验和，然后以校验和来引用。这意味着不可能在Git不知情时更改任何文件内容或目录内容。这个功能构建在Git底层，是构成Git哲学不可或缺的部分。若你在传送过程中丢失信息或损坏文件，Git就能发现。

## Git一般只添加数据
&emsp;&emsp;你执行的Git操作，几乎只往Git数据库中增加数据。很难让Git执行任何不可逆的操作，或者让它以任何形式清除数据。同别的VCS一样，未提交更新时有可能丢失或弄乱修改的内容，但是一旦你提交快照到Git中，就难以再丢失数据，特别是如果你定期的推送数据库到其它仓库的话。

# Git 的工作流程
## 先来了解4个专有名词
- Workspace（工作区）：
&emsp;&emsp;我们平时进行开发改动的地方，是我们当前看到的，也是最新的。平常我们开发就是拷贝远程仓库中的一个分支，基于该分支进行开发，在开发过程中就是对工作区的操作。

- Index / Stage（暂存区）：
&emsp;&emsp;.git目录下的index文件，暂存区会记录``` git add ``` 添加文件的相关信息（文件名、大小、timestamp...），不保存文件实体，通过id指向每个文件实体。可以使用 ``` get status ``` 查看暂存区的状态。暂存区标记了你当前工作区中，哪些内容是被git管理的。
&emsp;&emsp;当我们完成某个需求或功能后需要提交到远程仓库，那么第一步就是通过 ``` git add ``` 命令先提交到暂存区，被git管理。

- Local Repository（本地仓库）：
&emsp;&emsp;保存了对象被提交过得各个版本，比起工作区和暂存区的内容，它要更旧一些。``` git commit ``` 后同步index（暂存区）的目录树到本地仓库，方便从下一步通过 ``` git push ``` 同步本地仓库与远程仓库。

- Remote Repository（远程仓库）：
&emsp;&emsp;远程仓库是指托管在一些Git代码托管平台上的你的项目的版本库，比如[GitHub](https://github.com/)、[GitLab](https://about.gitlab.com/)、[码云](https://gitee.com/)、[码市](https://coding.net/)等等。远程仓库的内容可能被分布在多个地点的处于协作关系的本地仓库修改，因此它可能与本地仓库同步，也可能不同步，但是远程仓库的内容是最旧的。

&emsp;&emsp;四个区域之间的关系如下图所示：
![](https://ws1.sinaimg.cn/large/749c46aagy1fxx2bz7s91j20p20ivdhh.jpg)

# 常用的Git命令详解
- HEAD
&emsp;&emsp;HEAD，它始终指向当前所处分支的最新的提交点，所处的分支发生了变化，或者产生了新的提交点，HEAD就会跟着改变。
![](https://ws1.sinaimg.cn/large/749c46aagy1fxx4b6zt44j20f40ah0tc.jpg 'HEAD')

- git add
&emsp;&emsp;``` git add ``` 主要实现将工作区修改的内容提交到暂存区，交由git管理。相关命令如下表所示：

| 命令 | 说明 |
| :-: | :-: |
| git add . | 添加当前目录的所有文件到暂存区 |
| git add [dir] | 添加指定目录到暂存区，包括子目录 |
| git add [file] | 添加指定文件到暂存区 |
![](https://ws1.sinaimg.cn/large/749c46aagy1fxx4ana2tij20cr06s0t0.jpg 'git add')

- git commit
&emsp;&emsp;``` git commit ```主要实现将暂存区的内容提交到本地仓库，并使得当前分支的HEAD向后移动一个提交点。相关命令如下表所示：

| 命令 | 说明 |
| :-: | :-: |
| git commit -m [message] | 提交暂存区到本地仓库，message代表说明信息 |
| git commit [file] -m [message] | 提交暂存区的指定文件到本地仓库 |
| git commit —amend -m [message] | 使用一次新的commit，代替上一次提交 |
![](https://ws1.sinaimg.cn/large/749c46aagy1fxx4anfmlzj20f009x0t9.jpg 'git commit')

- git branch
&emsp;&emsp;在我们的代码仓库中，有一条主分支Master，我们可以从主分支当中，创建出许多分支以开发其他功能，创建子分支的好处是每个分支互不影响，大家只需要在自己的分支上继续开发，正常工作。待开发完毕后，再将自己的子分支合并到主分支或者其他分支即可，这样，即安全又不影响他人的工作。
&emsp;&emsp;关于分支，主要有展示分支、切换分支、创建分支、删除分支操作。相关命令如下表所示：

| 命令 | 说明 |
| :-: | :-: |
| git branch | 列出本地所有分支 |
| git branch -f | 列出远程所有分支 |
| git branch -a | 列出所有本地和远程的分支 |
| git branch [branch -name] | 新建一个分支，但依然停留在当前分支 |
| git branch —track [branch][remote-branch] | 新建一个分支，与指定的远程分支建立追踪关系 |
| git checkout -b [branch-name] | 新建一个分支，并切换到该分支 |
| git checkout [branch-name] | 切换到指定分支，并更新工作区  |
| git branch -d [branch-name] | 删除指定分支 |
| git push origin —delete [branch-name] | 删除远程分支 |
![](https://ws1.sinaimg.cn/large/749c46aagy1fxx6t7cb2bj20ot0bnmy9.jpg 'git branch')

- git merge
&emsp;&emsp;``` git merge ``` 命令的作用是把不同的分支合并起来。在实际的开发中，我们会从master主分支中创建出一个新的分支，然后进行需求或功能的开发，最后开发完成后需要合并子分支到主分支master中，这就需要用到 ``` git merge ```命令。

| 命令 | 说明 |
| :-: | :-: |
| git merge [branch] | 合并指定分支到当前分支 |
![](https://ws1.sinaimg.cn/large/749c46aagy1fxx76z1kc8j20l206sgm4.jpg 'git branch')

- git rebase
&emsp;&emsp;rebase又称为衍合，是合并的另外一种选择。在开始阶段，我们在新的分支上，执行 ``` git rebase dev ``` ，那么新分支上的commit都在master分支上重演一遍，最后checkout切换回到新的分支。这一点与merge命令是一样的，合并前后所处的分支并没有改变。
![](https://ws1.sinaimg.cn/large/749c46aagy1fxx7s107mvj20sd064dg5.jpg 'git rebase dev')

- git reset