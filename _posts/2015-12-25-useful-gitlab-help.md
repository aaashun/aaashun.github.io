---
layout: post
title: useful gitlab help
comments: true
tags: git github gitlab
---

当时为鼓动大家把项目迁到gitlab写的文档, 稍作修改, 分享出来

gitpages和gitlab从同一个md生成的html样式有差异, 原来排版没这么丑的

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [useful gitlab help](#useful-gitlab-help)
    - [设置git](#%E8%AE%BE%E7%BD%AEgit)
    - [快速入门](#%E5%BF%AB%E9%80%9F%E5%85%A5%E9%97%A8)
    - [工作流程](#%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B)
        - [develop阶段工作流程](#develop%E9%98%B6%E6%AE%B5%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B)
        - [release阶段工作流程](#release%E9%98%B6%E6%AE%B5%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B)
    - [常用操作](#%E5%B8%B8%E7%94%A8%E6%93%8D%E4%BD%9C)
        - [创建新分支](#%E5%88%9B%E5%BB%BA%E6%96%B0%E5%88%86%E6%94%AF)
        - [签出分支](#%E7%AD%BE%E5%87%BA%E5%88%86%E6%94%AF)
        - [列出所有branch](#%E5%88%97%E5%87%BA%E6%89%80%E6%9C%89branch)
        - [删除本地分支](#%E5%88%A0%E9%99%A4%E6%9C%AC%E5%9C%B0%E5%88%86%E6%94%AF)
        - [删除远程分支](#%E5%88%A0%E9%99%A4%E8%BF%9C%E7%A8%8B%E5%88%86%E6%94%AF)
        - [查看本地修改](#%E6%9F%A5%E7%9C%8B%E6%9C%AC%E5%9C%B0%E4%BF%AE%E6%94%B9)
        - [提交本地修改](#%E6%8F%90%E4%BA%A4%E6%9C%AC%E5%9C%B0%E4%BF%AE%E6%94%B9)
        - [推送到远程仓库](#%E6%8E%A8%E9%80%81%E5%88%B0%E8%BF%9C%E7%A8%8B%E4%BB%93%E5%BA%93)
        - [回滚](#%E5%9B%9E%E6%BB%9A)
        - [同步当前分支的更新](#%E5%90%8C%E6%AD%A5%E5%BD%93%E5%89%8D%E5%88%86%E6%94%AF%E7%9A%84%E6%9B%B4%E6%96%B0)
        - [同步其它分支的更新](#%E5%90%8C%E6%AD%A5%E5%85%B6%E5%AE%83%E5%88%86%E6%94%AF%E7%9A%84%E6%9B%B4%E6%96%B0)
        - [fork后同步上游仓库的更新](#fork%E5%90%8E%E5%90%8C%E6%AD%A5%E4%B8%8A%E6%B8%B8%E4%BB%93%E5%BA%93%E7%9A%84%E6%9B%B4%E6%96%B0)
        - [引用公共代码](#%E5%BC%95%E7%94%A8%E5%85%AC%E5%85%B1%E4%BB%A3%E7%A0%81)
    - [project管理](#project%E7%AE%A1%E7%90%86)
        - [create](#create)
        - [fork](#fork)
        - [permission](#permission)
        - [group](#group)
        - [transfer](#transfer)
    - [其它](#%E5%85%B6%E5%AE%83)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

<br/>
<br/>
# useful gitlab help
## 设置git
查看当前版本, v1.8以下的必须[升级](http://git-scm.com/downloads)

```sh
$ git --version
```

删除global用户信息, 防止与github或其它git库冲突

```sh
$ git config --global --unset user.name
$ git config --global --unset user.email
```

使用https访问, ssh不对外开放. 由于是自已签发的证书, 需要取消ssl验证. 另外为了避免每次都输入账号密码, 这里设置了3600秒的超时时间

```sh
$ git config --global http.sslverify false
$ git config --global credential.helper 'cache --timeout=3600'
```

配置可视化diff和merge工具, 在linux系统上建议使用meld

```sh
$ git config --global diff.tool meld
$ git config --global merge.tool meld
```

## 快速入门

clone仓库, 这里以aaashun/useful-gitlab-help.git为例, 只是示例, 没有权限提交代码

```sh
$ git clone https://gitlab.example.com/aaashun/useful-gitlab-help.git
```

设置local用户信息, 不推荐使用global的用户信息设置

```sh
$ git config --local user.name <your-name> #与邮箱的用户名保持一致
$ git config --local user.email <your-email>
````

创建branch, 并切换进去

```sh
$ git branch <feature-branch-name>
$ git checkout <feature-branch-name>
```

编写代码, 例如在README.md末尾增加一行

```sh
$ echo -e "\nxxx到此一游" >> README.md
```

commit到本地仓库

```sh
$ git commit -am "this is my first commit"
```

push到远程仓库

```sh
$ git push -u origin <feature-branch-name>
```

OK, 在web页面上应该可以看到提交的代码了
> sign in -> project -> useful-gitlab-help -> BRANCH -> feature-branch-name -> README.md


这里只是第一步将更新提交到feature分支, 下一章会细说如何将代码合并到master分支

## 工作流程

在之前我们使用svn时, 由于没有制定规范的work flow, 所以对trunk, branch, tag等使用上较乱, 切换到git后希望从一开始大家就有一个统一的work flow认识.

而git的branch使用太过灵活, 不同组织在使用时都会制定一个的work flow, 比如[git flow, github flow, gitlab flow](https://gitlab.example.com/help/workflow/gitlab_flow.md)

在git flow基础之上稍作修改, 建议如下git flow:

![gitflow](/img/gitflow.jpg)

上图work flow包含了研发的两个阶段, master左边的是常规develop阶段, master右边的是release阶段. 下边两小节分边对此祥细说明.

### develop阶段工作流程

master分支是受保护的, 只有master权限的成员才有权向master分支提交代码, 日常开发工作都是在feature分支完成.

一个feature分支仅用于完成一个功能特性或开发阶段的某个问题, 一个feature分支的生命周期尽量控制在几天时间, 时间长了在合并时会有痛苦的冲突.

feature分支的命名尽量用大白话, 让其它人看名称就知道这个分支是在完成什么功能或解决什么问题, 如下示例

```sh
$ git branch li.li                         # bad, do not use username as branch name
$ git branch new-java-bootloader           # good
$ git branch refactor                      # bad, what are you refactoring now? 
$ git branch refactor-processor-framework  # good
```

feature分支完成开发并经自测验证没问题时, 向master分支发起merge request, 在web页面上项目组其它成员review你的代码, 很多时候需要多次修改才通过code review, review通过后, 由项目管理员将代码merge到master分支
> sign in -> Projects -> enter your project -> Merge Requests -> NEW MERGE REQUEST

合并过的feature分支应该立即删除

`名言名句: git不是万能的, 没有code review是万万不能的`

### release阶段工作流程

包括gitlab flow和gitlab flow都认为master分支是稳定的分支, 随便一个版本都可以发布的, 我认为人少的项目还比较好控制, 在多人的大项目里, 很多时候需要经过大量的黑盒测试工作才能发布版本.

OK, 经过一段时间的开发, 完成了计划的功能, master上的代码也通过更严格的测试了, 可以对外发布了. 每次发布需要打一个tag, tag名称即是版本号, 例如v0.1.0

使用一段时间后收集到一些问题需要在v0.1.0版本上修复, 这时需要创建一个release分支v0.1.x, 在v0.1.x上完成bug修复, 并发布v0.1.1, v0.1.2等bugfix版本

## 常用操作

这里只是流水账似的列出常用的几个操作, 要想真正用得熟练, 还得[多看文档掌握重要的git命令如branch, checkout, push, pull, merge, fetch, reset等](http://git-scm.com/docs)

### 创建新分支

```sh
$ git branch <branch-name>
```

### 签出分支
也叫切换分支

```sh
$ git checkout <branch-name>
```

### 列出所有branch
标星号的是当前branch

```sh
$ git branch -a 
```

### 删除本地分支

```sh
$ git branch -d <local-branch-name>
```

### 删除远程分支 

```sh
$ git push origin --delete <remote-branch-name>
```

### 查看本地修改

```sh
$ git diff
$ git difftool  # diff和difftool都可以
```

### 提交本地修改

```sh
$ git commit -m "commit log" <file-pattern> # 提交部分
$ git commit -m "commit log" -a             # 提交所有
```

### 推送到远程仓库

```sh
$ git push origin <branch-name>
```

### 回滚
恢复到某个文件到最近一次提交的版本/取消某个文件的本地修改

```sh
$ git checkout <file-name>
```

恢复某个文件到历史版本

```sh
$ git log <file-name>                   # 查看日志中的commit-number
$ git reset <commit-number> <file-name>
$ git checkout <file-name>
```

### 同步当前分支的更新
例如当前feature分支还在开发中, 其它小伙伴向当前提交了代码\
`不建议使用pull命令, 因为它隐藏了一些细节, 这里用fetch & merge命令替代, 每一步心里有数`

```sh
$ git fetch origin <branch-name>              # 下载最新的代码到远程跟踪分支, 即origin/<branch-name>
$ git difftool <branch-name> origin/<branch-name> # 查看更新内容, 看看而已, 不要在这一步执行手工merge
$ git merge origin/<branch-name>              # 尝试合并远程跟踪分支的代码到本地分支
$ git mergetool                               # 借助mergetool解决冲突
```

### 同步其它分支的更新
例如当前feature分支还在开发中, master上已经有了更新的代码, 和上例同步当前分支更新的步骤类似

```sh
$ git fetch origin master
$ git difftool <branch-name> origin/master
$ git merge origin/master
$ git mergetool
```

### fork后同步上游仓库的更新
如下示例里假设你fork了aaashun/useful-gitlab-help, 你想享受到上游的更新

```sh
$ git remote -v # 查看远程仓库
origin https://gitlab.example.com/li.li/useful-gitlab-help.git (push)
origin https://gitlab.example.com/li.li/useful-gitlab-help.git (fetch)

$ git remote add upstream https://gitlab.example.com/aaashun/useful-gitlab-help.git # 第一次需要添加上游仓库
$ git remote -v
origin  https://gitlab.example.com/li.li/useful-gitlab-help.git (push)
origin  https://gitlab.example.com/li.li/useful-gitlab-help.git (fetch)
upstream    https://gitlab.example.com/aaashun/useful-gitlab-help.git (push)
upstream    https://gitlab.example.com/aaashun/useful-gitlab-help.git (fetch)


$ git fetch upstream # 和上例同步当前分支更新的步骤类似了
$ git difftool <branch-name> upstream/master
$ git merge upstream/master
$ git mergetool
```

### 引用公共代码
代码引用在git上有两种方式: submodule和subtree, 推荐使用subtree方式, submodule需要每个人都有公共项目的权限, 而subtree只需要一个有权限访问公共库的人来维护就行了, 对团队其它成员是透明的\
`虽然可以直接引用代码库, 但是基础项目尽量对外发布二进制库, 而不是开放源代码给其它项目直接引用`

```sh
$ # 第一次初始化
$ git remote add -f <remote-subtree-repository> <remote-subtree-repository-url>
$ git subtree add --prefix=<local-directory> <remote-subtree-repository> <remote-subtree-branch-name> --squash

$ # 同步subtree的更新
$ git fetch <remote-subtree-repository> <remote-subtree-branch-name>
$ git subtree pull --prefix=<local-directory> <remote-subtree-repository> <remote-subtree-branch-name>
```

## project管理
### create
在网页上创建, 默认限制一个用户最多创建3个项目, 如需创建更多项目请与[管理员]联系

### fork
只有当你有某个项目的读权限但是没有写权限时才需要fork, 这在开源界用的多, 但在公司内部基本上都是private项目, 所以使用不多

### permission
所有项目默认是private, 即只对该项目组内的成员开放权限, 项目的owner(默认是创建者)负责项目的访问权限设置, 并且可以对每一个成员设置权限

### group
有时一个项目比较大, 会有很多子项目, 这时可以专门为这个大项目创建一个分组, 例如xxxx分组下有xxxx-core, xxxx-for-car, xxxx-for-phone等project

如需创建group请与[管理员]联系

### transfer

可以将一个项目过户给另一个用户, 比如将 aaashun 创建的 [useful-gitlab-help] 过户给 li.li, 过户后的地址就变成了 https://gitlab.example.com/li.li/useful-gitlab-help

## 其它
[dillinger] 是优秀的markdown在线编辑器

[//]: # (reference links)
[useful-gitlab-help]: https://gitlab.example.com/aaashun/useful-gitlab-help
[dillinger]: http://dillinger.io
[管理员]: mailto:admin@gitlab.example.com
