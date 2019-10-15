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
&emsp;&emsp;在我们日常的开发工作中，对代码的管理一般会用到Git，当然也有使用SVN。对于刚接触Git或者仅仅是使用`git pull`、`git push`等几个常用命令的人来说，对Git还是挺陌生的，不理解其原理和如何具体使用。本文会对Git的原理以及常用命令进行介绍，希望对刚接触Git的开发者有一定的帮助。<!-- more -->
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxy91cj41qj208c05iwf8.jpg 'Git')

# Git基础和原理
## 直接记录快照，而非差异比较
&emsp;&emsp;Git和其他版本控制系统（包括SVN和近似工具）的主要差别在于Git对待数据的方法。概念上来区分，其他大部分系统以文件变更列表的方式存储信息。这类系统（CVS、Subversion、Perforce、Bazaar等等）将它们保存的信息看作是一组基本文件和每个文件随时间逐步累积的差异。Git不按照以上方式对待或保存数据。反之，Git更像是把数据看作是对小型文件系统的一组快照。每次你提交更新，或在Git中保存项目状态时，它主要对当时的全部文件制作一个快照并保存这个快照的索引。为了高效，如果文件没有修改，Git不再重新存储该文件，而是只保留一个链接指向之前存储的文件。Git对待数据更像是一个快照流。如下图所示。这是Git与几乎所有其它版本控制系统的重要区别。
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxwutjhlfrj20m808hwgu.jpg)
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
&emsp;&emsp;.git目录下的index文件，暂存区会记录`git add`添加文件的相关信息（文件名、大小、timestamp...），不保存文件实体，通过id指向每个文件实体。可以使用`git status`查看暂存区的状态。暂存区标记了你当前工作区中，哪些内容是被git管理的。
&emsp;&emsp;当我们完成某个需求或功能后需要提交到远程仓库，那么第一步就是通过 `git add`命令先提交到暂存区，被git管理。

- Local Repository（本地仓库）：
&emsp;&emsp;保存了对象被提交过的各个版本，比起工作区和暂存区的内容，它要更旧一些。`git commit`后同步index（暂存区）的目录树到本地仓库，方便从下一步通过`git push`同步本地仓库与远程仓库。

- Remote Repository（远程仓库）：
&emsp;&emsp;远程仓库是指托管在一些Git代码托管平台上的你的项目的版本库，比如[GitHub](https://github.com/)、[GitLab](https://about.gitlab.com/)、[码云](https://gitee.com/)、[码市](https://coding.net/)等等。远程仓库的内容可能被分布在多个地点的处于协作关系的本地仓库修改，因此它可能与本地仓库同步，也可能不同步，但是远程仓库的内容是最旧的。

&emsp;&emsp;四个区域之间的关系如下图所示：
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxx2bz7s91j20p20ivdhh.jpg)

# 常用的Git命令详解
## HEAD
&emsp;&emsp;HEAD，它始终指向当前所处分支的最新的提交点，所处的分支发生了变化，或者产生了新的提交点，HEAD就会跟着改变。
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxx4b6zt44j20f40ah0tc.jpg 'HEAD')


## git add
&emsp;&emsp;`git add`主要实现将工作区修改的内容提交到暂存区，交由git管理。相关命令如下表所示：

| 命令 | 说明 |
| :-: | :-: |
| git add . | 添加当前目录的所有文件到暂存区 |
| git add [dir] | 添加指定目录到暂存区，包括子目录 |
| git add [file] | 添加指定文件到暂存区 |
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxx4ana2tij20cr06s0t0.jpg 'git add')

## git commit
&emsp;&emsp;`git commit`主要实现将暂存区的内容提交到本地仓库，并使得当前分支的HEAD向后移动一个提交点。相关命令如下表所示：

| 命令 | 说明 |
| :-: | :-: |
| git commit -m [message] | 提交暂存区到本地仓库，message代表说明信息 |
| git commit [file] -m [message] | 提交暂存区的指定文件到本地仓库 |
| git commit —amend -m [message] | 使用一次新的commit，代替上一次提交 |
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxx4anfmlzj20f009x0t9.jpg 'git commit')

## git branch
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
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxx6t7cb2bj20ot0bnmy9.jpg 'git branch')

## git merge
&emsp;&emsp;`git merge`命令的作用是把不同的分支合并起来。在实际的开发中，我们会从master主分支中创建出一个新的分支，然后进行需求或功能的开发，最后开发完成后需要合并子分支到主分支master中，这就需要用到`git merge`命令。

| 命令 | 说明 |
| :-: | :-: |
| git merge [branch] | 合并指定分支到当前分支 |
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxx76z1kc8j20l206sgm4.jpg 'git branch')

## git rebase
&emsp;&emsp;rebase又称为衍合，是合并的另外一种选择。在开始阶段，我们在新的分支上，执行`git rebase dev`，那么新分支上的commit都在master分支上重演一遍，最后checkout切换回到新的分支。这一点与merge命令是一样的，合并前后所处的分支并没有改变。
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxx7s107mvj20sd064dg5.jpg 'git rebase dev')

## git reset
&emsp;&emsp;`git reset`命令把当前分支指向另一个位置，并且有选择的变动工作目录和索引。也用来从历史仓库中复制文件到索引，而不动工作目录。如果不给选项，那么当前分支指向到那个提交。如果用--hard参数，那么工作目录也更新，如果用--soft参数，那么都不变。使用 `git reset HEAD ~3` 命令的说明如下图所示：
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxy00bvqa4j20mk0c9wfo.jpg 'git reset HEAD~3')

&emsp;&emsp;如果没有给出提交的版本号，那么默认用HEAD。这样，分支指向不变，但是索引会回滚到最后一次提交，如果用--hard参数，工作目录也一样。
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxy00byt7bj20o50c43zh.jpg 'git reset')


| 命令 | 说明 |
| :-: | :-: |
| git reset —soft [commit] | 只改变提交点，暂存区和工作目录的内容都不改变 |
| git reset —mixed [commit] | 改变提交点，同时改变暂存区的内容 |
| git reset —hard [commit] | 暂存区、工作区的内容都会被修改到与提交点完全一致的状态 |
| git reset —hard HEAD | 让工作区回到上次提交时的状态 |

## git revert
&emsp;&emsp; revert 是用一个新的提交来消除一个历史提交所做的任何修改，revert之后，我们本地的代码会回滚到指定的历史版本。举个🌰，其结果如下图所示：
```
git commit -am 'update readme'
git revert 15df9b6
```
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxy16i1trqj20ez06nt90.jpg 'git revert')

## git push
&emsp;&emsp;将本地仓库分支上传到远程仓库分支，实现同步。相关命令如下：

| 命令 | 说明 |
| :-: | :-: |
| git push [remote] [branch] | 上传本地指定分支到远程仓库 |
| git push [remote] —force | 强行推送当前分支到远程仓库，即使有冲突 |
| git push [remote] —all | 推送所有分支到远程仓库 |

## git fetch
&emsp;&emsp;`git fetch`是从远程仓库中获取最新版本到本地仓库中，但不会自动合并本地的版本，也就说，我们可以查看更新情况，然后再决定是否进行合并。在fetch命令中，有一个重要的概念：**FETCH_HEAD**：某个branch在服务器上的最新状态，每一个执行过fetch操作，都会存在一个FETCH_HEAD列表，这个列表保存在**.git**目录的**FETCH_HEAD**文件中，其中每一行对应于远程服务器的每一个分支。当前分支指向的FETCH_HEAD，就是文件第一行对应的那个分支。

## git pull
&emsp;&emsp;`git pull`是从远程仓库中获取最新版本并自动合并到本地的仓库。
```
git pull origin next
```
&emsp;&emsp;上面命令表示，取回**origin/next**分支的更新，再与当前分支进行合并。实质上，等同于，先做`git fetch`，再执行`git merge`。
```
git fetch origin
git merge origin/next
```

# 一些命令的区别
## merge 和 rebase
&emsp;&emsp;举个🌰说明：现在我们有这样两个分支，test和master，其提交记录如下图所示：
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxy73k6ut8j20ca04z74f.jpg)
&emsp;&emsp;在master分支上执行`git merge test`命令后，会得到如下图所示的结果：
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxy73k8nxxj20fp04s74h.jpg)
&emsp;&emsp;如果在master分支上执行`git rebase test`命令，则会得到如下图所示的结果：
![](http://pz1livcqe.bkt.clouddn.com/749c46aagy1fxy73kcohsj20hc03mt8t.jpg)
&emsp;&emsp;由上面的🌰可以看出，merge操作会生成一个新的节点，之前的提交分开显示。而rebase操作不会生成新的节点，是将两个分支融合成一个线性的提交记录。
&emsp;&emsp;如果我们想要一个干净的，没有merge commit的线性历史树，那么应该选择`git rebase`，如果想保留完整的历史记录，并且想要避免重写commit history的风险，应该选择使用`git merge`。

## reset 和 revert
- revert 是用一次新的commit来回滚之前的commit，而 reset 是直接删除指定的commit。
- 在回滚这一操作上看，效果差不多。但在日后继续merge以前的老版本时有区别。因为`git revert`是用一次逆向的commit“中和”之前的提交，因此日后合并老的分支时，导致这部分改变不会再次出现，减少冲突。但是`git reset`是直接把某个commit在某个分支上删除，因而和老的分支再次merge时，这些被回滚的commit应该还会被引入，产生很多冲突。
- `git reset`是把HEAD向后移动了一下，而`git revert`是HEAD继续前进，只是新的commit的内容和要revert的内容正好相反，能够抵消要被revert的内容。
    
## fetch 和 pull
&emsp;&emsp;`git fetch`和`git pull`共同点都是从远程的分支获取最新的版本到本地，但fetch命令不会自动将更新合并到本地的分支，而pull命令会自动合并到本地的分支。在实际的使用中，`git fetch`要更安全一些，因为获取到最新的更新后，我们可以查看更新情况，然后再决定是否合并。
    
# 参考资料
- [git官网](https://git-scm.com/docs)
- [易百教程 - Maxsu - Git教程](https://www.yiibai.com/git)
- [掘金 - Ruheng - 一篇文章，教你学会Git](https://juejin.im/post/599e14875188251240632702)
- [廖雪峰的官方网站 - 廖雪峰 - Git教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)
- [CSDN - hudashi - git fetch 和 git pull的区别](https://blog.csdn.net/hudashi/article/details/7664457)
- [博客园 - Yelbosh - Git的原理简介和常用命令](https://www.cnblogs.com/yelbosh/p/7471979.html)