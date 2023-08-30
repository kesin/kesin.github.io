---
layout: post
title: 一起来玩 Jenkins:（1）基础搭建和功能介绍
date: 2020-08-22 04:19:24
excerpt: 一起来玩 Jenkins 系列主要是自己对 Jenkins 使用的见解和总结，将从 Jenkins 的基础安装、功能介绍等入手，外加实际使用的 Demo 配置流程，使大家能够了解 Jenkins 的基本功能和实际应用，希望能帮助到对它感兴趣的朋友。
---

### Jenkins 简介
Jenkins是一款开源 CI&CD 软件，用于自动化各种任务，包括构建、测试和部署软件，支持各种运行方式，可通过系统包、Docker 或者通过一个独立的 Java 程序。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/104126_b678934b.png "在这里输入图片标题")

> 摘自 Jenkins 中文手册 https://www.jenkins.io/zh/doc/

从开始接触企业级大客户招商证券、招商银行、比亚迪、华为等企业后，越来越多的有接触到 Jenkins 的使用需求，也越来越多的开始关注 DevOps 为一个企业或者大型团队带来的效率提升，一个企业或者持续交付的项目越是往后发展，越会对自动化工程化的平台需求愈发强烈，因为它能够给整个研发链带来非常明显的效率和质量的提升，其实总的来讲，我个人觉得大家可以简单的把 DevOps 理解为「能给自动化的全自动化了，能不人工介入的就不人工介入」，而 Jenkins 就是帮助我们进行自动化的工具。

好了，下面我们开始探索 Jenkins 的世界吧！

### 下载和安装

访问 https://www.jenkins.io/download/ 下载最新版本的 Jenkins

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/104403_82e4146f.png "在这里输入图片标题")

Jenkins 的安装方式多种多样，对于初识 Jenkins 的朋友，建议使用最简单的 `war` 包的安装方式进行安装，只需要有 Java 环境即可，需要注意的是Jenkins 最新版本对 Java 要求：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/104507_21cc93b5.png "在这里输入图片标题")

下载最新版本的 [JDK](https://download.oracle.com/otn/java/jdk/8u261-b12/a4634525489241b9a9e1aa73d9e118e6/jdk-8u261-linux-x64.tar.gz) 配置安装升级即可 （下载需要注册 Oracle 账号）

### 启动

通过命令 `java -jar jenkins.war` 即可启动 Jenkins，默认的绑定端口是`8080`，我们也可以通过追加`--httpPort=8080`来指定 Jenkins 端口，启动后可以通过`127.0.0.1:8080`访问
> Tip1：Jenkins 启动后会在我们的`$HOME`目录创建`.jenkins`运行目录  
> Tip2：如果是通过其他机器无法访问，请检查端口是否开放  
> Tip3：如果需要后台运行，可以使用 `nohup java -jar jenkins.war &`  

执行启动命令后，Jenkins 会进行一些初始化的操作，当出现这个界面，说明 Jenkins 初始化完毕可以进行访问了

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/105411_24853e6c.png "在这里输入图片标题")

### 初始化

在浏览器中输入`127.0.0.1:8080`访问 Jenkins

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/105617_721051b9.png "在这里输入图片标题")

Jenkins 在首次访问的时候会要求我们输入初始的密码，这是为了防止如果我们部署在公网被其它人误操作的情况，密码可以在启动的日志中找到，也可以在 Jenkins 的工作目录找到：
```ruby
/home/zoker/.jenkins/secrets/initialAdminPassword
```
输入密码后，我们点击 Continue 继续

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/110143_9f736ec4.png "在这里输入图片标题")

此时 Jenkins 要求我们选择一些插件，我们选择推荐的插件即可，如果对 Jenkins 已经比较熟悉，那么可以选择自己所需要的插件进行安装，这些插件在后续都可以在插件中心进行安装和卸载，所以这里选择推荐插件即可

> Jenkins 插件为 Jenkins 提供了丰富多样的功能，完善了 Jenkins 的生态，这也是 Jenkins 如此流行的重要原因之一。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/110438_00cb4ca4.png "在这里输入图片标题")

点击`Install suggested plugins`后，Jenkins 就开始安装推荐的插件，我们可以看到包含了一些基础的插件功能如时间戳、Git、Email 邮件以及 Ant、Gradle等构建工具。整个安装的时间取决于机器的网络状况，耐心等待即可，也可以中途中断，下次启动 Jenkins 也会继续进行增量的安装。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/110800_5ae2fe52.png "在这里输入图片标题")

插件安装完毕后，Jenkins 会自动跳转到用户创建的界面，为了安全性保证，这里建议大家都创建一个用户，创建完成后，点击 `Save and Continue`

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/110949_4f67c164.png "在这里输入图片标题")

Admin 用户初始化完毕后，Jenkins 会要求我们确认 Jenkins URL，这个 URL 就是我们安装的 Jenkins 的根域名，如果后续我们对外开放是使用域名或者其它方式，这里应该替换掉，因为它会出现在通知、系统回调等内容里。

这里我们不做修改，点击 `Save and Finish` 即可完成 Jenkins 的初始化安装

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/111236_cc21a15b.png "在这里输入图片标题")

点击 `Start using Jenkins` 即可进入 Jenkins 的世界！

### 功能简介

Jenkins 的初始界面给人的第一感觉是简单，但是当你用下去之后你会发现它并不是看上去的那么简单，功能强大且丰富。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/111345_5d07daf5.png "在这里输入图片标题")

左侧栏主要是常用功能展示
```
New Item             用来创建自定义任务或者流水线的入口
People               用来管理 Jenkins 上产生的各种用户
Build History        用来查看所有的构建历史
Manage Jenkins       主要功能，Jenkins 的管理入口，所有的配置和插件都在此处进行管理
My Views             查看自己的自定义视图
Lockable Resources   资源锁定，比如一台构建机器一次只想执行一个`Job`，其它的`Job`则排队等待释放
New Views            创建自定义视图
```
这里我们主要介绍创建流水线的类型和插件安装介绍

进入到`New Item`，此处是创建流水线任务的入口

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/113946_31b2dd33.png "在这里输入图片标题")

**Freestype project** 是一个灵活的自由项目，你可以用它来灵活的配置一些项目而不受到太多约束

**Pipeline** 需要我们对 `Pipeline Script` 的编写有一定的经验，它可以在多个构建机器上进行执行，并且可以并行的执行任务

**Multi-configuration project** 适合需要大量配置化工作的项目，比如需要在多个不同的环境跑测试、构建等工作

**Folder** 类似于一个独立的命名空间，可以包含多个不同类型的流水线任务

**GitHub Organization** 是安装了相关的默认插件才有的选项，主要用来对 Github 组织下的所有仓库进行扫描和导入（也计划给 Gitee 做一个）

**Multibranch Pipeline** 可以导入一个仓库，并且会自动的对仓库内的所有分支进行自动发现可流水线的创建，主要依赖于 `Jenkinsfile`

进入到 `Manage Jenkins` 此处是 Jenkins 的管理界面

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/115423_689e3625.png "在这里输入图片标题")

其它的不做过多的介绍，后续用到会进行解释，这里我们来看看 Jenkins 中最重要的插件的管理，我们进入到 `Manage Plugins`

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/115600_49dfff72.png "在这里输入图片标题")

一共有四列：有更新、可用的、已安装、配置，我们可以在这里管理我们所有的 Jenkins 插件，下面以安装一个 Jenkins 汉化插件为例介绍

我们在 `Available`下输入`chinese`，可以看到由 Jenkins 中文社区维护的一个中文插件

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/115915_3418aceb.png "在这里输入图片标题")

勾选并点击`Install without restart`

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/120035_17fa0070.png "在这里输入图片标题")

自动进入到下载安装过程，我们勾选底部的 `Restart Jenkins when installation is complete and no jobs are running`，让 Jenkins 在没有任务运行的时候自动重启使安装的插件生效

> 如果插件下载有网络问题，可以在`Advanced`里面配置代理或者用上传插件的方式进行安装

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0822/121147_a00211e3.png "在这里输入图片标题")

Jenkins 自动重启后即可看到中文已经生效

> Tips1：自动重启后如果看到还是英文，不要着急，说明你的浏览器环境是英文，将系统语言设置为中文即可。  
> Tips2：如果不能自动重启，我们可以通过在 Jenkins URL 后加 `/restart` 来手工进行重启：http://127.0.0.1:8080/restart

至此，Jenkins 已经完全安装并配置好，可以开始投入工作啦！

**下篇预告：新手入门实例操作指南 - Hello Jenkins！**