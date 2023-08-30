---
layout: post
title: 一起来玩 Jenkins:（2）入门实例及 Pipeline 介绍与使用 
date: 2020-09-05 16:23:11
excerpt: 上篇中我们主要介绍了 Jenkins 的安装以及基本的一些配置和插件的使用，本章节主要从流水线层面介绍下不同类型的流水线的使用，并来介绍下 Jenkins Pipeline 及 Jenkinsfile 基本语法。
---

本篇主要从两种不同的 Jenkins 项目来初步了解下 Jenkins

### 1、Freestyle project

上节讲到 Freestyle project 是一个灵活的配置项目，就像它的英文名字一样，我们使用它来实现我们第一个 Jenkins 项目并在终端打印出 `Hello Jenins!`

首先，点击主界面左上角的`New Item`

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/154847_d8b74812.png "在这里输入图片标题")

选择 Freestyle project 并输入名字

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/155114_c2558fc4.png "在这里输入图片标题")

点击保存后，我们就进入到了项目的配置界面

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/155451_5c90a6d0.png "在这里输入图片标题")

可以看到 Freestyle 项目有如下几个配置选项：

- 常规，用来配置一些描述，是否需要锁定的资源等等
- 源码，用来配置源码仓及源码仓相关的权限等配置
- 构建触发，配置如何触发构建，比如通过webhook、通过定时检测版本等
- 构建环境，配置一些构建过程相关的选项，比如生成版本日志、增加时间戳到输出面板等
- 构建，用来配置构建过程，比如执行shell等
- 构建后，用来配置构建工作完成后需要做什么，比如打包制品、发布报告、提交任务或者通过各种方式通知等

我们来针对几个关键的点做下简单的配置，首先是构建触发器，可以选择远程触发，我们可以配置一个 Token 来增强它的安全性

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/173510_054e6734.png "在这里输入图片标题")

Jenkins 会生成一个地址，这个地址需要带上我们配置的 Token ，只要地址被访问调用，Jenkins 就会被触发构建，这里生成的地址是：
```
127.0.0.1:8080/job/MyFristProject/build?token=test111
```
然后我们配置下 Build 过程中需要做什么事情，这里选择执行一个脚本

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/173907_0e4dfde5.png "在这里输入图片标题")

当开始构建的时候，Jenkins 会执行脚本打印出`Hello Jenkins!`

点击保存，会自动跳转到项目的首页，这个时候任务还没有开始执行，我们需要点击左侧菜单栏的 Build Now

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/174127_59ea8653.png "在这里输入图片标题")

然后稍等片刻就可以在左下角看到刚刚手动执行过的一个任务

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/174214_c636dadf.png "在这里输入图片标题")

点击进入任务详情，并点击详情左侧的 Console Output，可以看到我们所期望的输出

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/174322_ab4d7282.png "在这里输入图片标题")

日常开发的过程中，我们不可能手动去触发构建操作的，这个构建操作一定是在代码提交后或者PR合并后进行触发的，一方面我们可以使用上面介绍到的自动定时检测版本库更新，另一方面我们可以使用上面设置的钩子地址，在 Gitee 或者 Github 上配置这个地址，并设置触发的场景，当我们代码有更新，平台会自动请求这个地址触发构建，这里我们手动访问下地址看下效果：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/174956_2f70c55e.png "在这里输入图片标题")

可以看到已经被正确触发了，显示是远程主机触发的，至此，一个基本的 Jenkins 的 Demo 已经完成，我们通过对 Jenkins 的设置，完成了对`Hello Jenkins!`的输出，相同的道理，脚本我们可以加上编译构建甚至其它可以做的事情。

接下来我们来看看 Jenkins 的 Pipeline

### 2、Pipeline

刚接触 Jenkins 的同学可能会有疑问，已经有了 Freestyle Project 了，我们可以在shell脚本里面写任何我们想要做的操作，为什么还要有个 Pipeline？

其实这个主要还是看场景，大部分的工程都可以使用前者来实现，但是配置起来比较麻烦，而且无法像 Pipeline 一样使用版本进行控制，也无法方便的实现多主机操作，Pipeline 更像一种规范，定义了模块参数及执行的过程，但是 Pipeline 需要了解其脚本语法基础，相对来说有一些额外的学习成本，下面是一个简单的 Pipeline 脚本

```
pipeline{
    agent any    // 定义 Pipeline 在哪台主机上运行
    stages{        // 任务区域
        stage('build'){     // 任务阶段
            steps{    // 任务步骤
                sh 'echo "Hello Jenkins!"'
            }
        }
    }
}
```

脚本定义了一个任务，输出`Hello Jenkins!`字符，其中我们可以在一个任务区域内，定义多个任务阶段，每个阶段可以定义多个步骤，而且这些都是可以设置是否并行的，这比使用脚本去实现着实简单多了吧？ :)

好了，我们来建立一个 Pipeline 试试看

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/180402_88fa9322.png "在这里输入图片标题")

创建成功后，也会让我们进行配置

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/180444_32d1c929.png "在这里输入图片标题")

可以看到配置项相较于 Freestyle 的明显变少了，这是因为很多事情都可以使用 Pipeline 的语法来做，更容易管理，我们直接跳到 Pipeline 的设置项去，可以选择是 Pipeline 脚本还是从版本控制选择 Jenkinsfile 作为脚本，为了演示方便，我们直接选择 Pipeline 即可

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/181044_8eaf737d.png "在这里输入图片标题")

把刚刚实例的脚本填进去，并点击保存，我们来像刚刚一样手动出发一个构建

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/181315_82592a35.png "在这里输入图片标题")

可以看到，日志比 Freestyle 的要清晰多了，每一个阶段都有标示输出，我们打开项目的首页，可以看到多了一个`Stage View`，可以很容易的看出每一次构建不同阶段的耗时，是不是觉得很方便？ :)

如果你觉得 Pipeline 语法还不是太会，Jenkins 也提供了生成器，可以通过生成器来生成这些脚本，比如我来生成一个读取文件的脚本

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0915/181726_5cf18179.png "在这里输入图片标题")

是不是很方便？ :)

### 总结

Jenkins Pipeline 帮我们处理了很多其它额外的工作，让我们能够专注于实际应用的脚本的编写，并且通过统一的脚本配置可以方便的通过版本控制管理和迁移过程，其它类型如`Multi-configuration project`，基本都是基于 Pipeline 的扩展类型，就不再赘述了，下一篇将会介绍下新一代的 Jenkins 流水线管理利器 - Blue Ocean