---
layout: post
title: 一起来玩 Jenkins:（3）BlueOcean 介绍与使用 
date: 2020-09-17 15:54:22
excerpt: 上篇中我们介绍了 Pipeline 的语法，也可以通过 Pipeline Editor 进行编辑，但是这些还不够，Jenkins 团队为了用户更方便的使用流水线，降低用户使用成本以及优化用户使用体验，推出了 BlueOcean，BlueOcean 是以插件的形式存在的，用官方的话说就是重新思考用户体验，它提供了一个更加灵活的流水线编辑器，允许你更加直观的进行流水线的定制，更加直观的观察到每一个步骤甚至每一个任务的运行状况。
---

### 安装

BlueOcean 的安装非常简单，它是以插件的集合的形式提供所有功能，我们打开 Jenkins 的插件管理，搜索 BlueOcean，就可以看到这个插件，点击单选框选中它，我们可以看到很多其它的插件也默认被选中了，因为正是由这些插件组成的 BlueOcean 强大的功能

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/193441_5776d435.png "在这里输入图片标题")

安装好之后我们返回 Jenkins 控制台，在左侧栏菜单就可以看到 BlueOcean 已经安装好了

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/193928_aa996b2b.png "在这里输入图片标题")

### 使用

点击 BlueOcean 进入控制台

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/194208_6a6c6e54.png "在这里输入图片标题")

可以看到已经进入了全新的 BlueOcean 视图，如果你之前已经创建过 FreeStyle 或者 Pipeline 等类型的流水线，这里也同样会展示，但是由于类型不同，可能会不可以修改或者进入流水线编辑器，这里我们点击新建流水线，通过一个全新的流水线项目来展示如何使用 BlueOcean

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/195536_c2318a1e.png "在这里输入图片标题")

点击选择 Github 后，需要我们输入 Github 的私人令牌，根据指引我们前往 Github 上进行创建（别问我为什么不用 Gitee，因为这里没有提供 Gitee 的选项，后面我会提个插件来支持，怎么可能没有 Gitee 呢？XD）

![1](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/195654_b5b7e9b0.png "在这里输入图片标题")

将生成的私人令牌填写到 Jenkins 并点击 Connect

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/195935_b776aad0.png "在这里输入图片标题")

选择你想要的仓库的命名空间，这里我刚刚在自己的名下创建了一个 BlueOceanTest 的仓库，我们选择 kesin 并选择 BlueOceanTest 仓库，点击 Create Pipeline，稍等片刻就会自动进入到流水线编辑器，如果你的仓库没有 Jenkinsfile，Jenkins 会提示你没有 Jenkinsfile，将会为你自动创建一个

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/200146_38ead203.png "在这里输入图片标题")

这里就已经进入到了流水线编辑器，左侧是可视化流水线编辑器，右侧是对每一个 Stage 或者 Steps 进行编辑的界面，我们点击左侧的+号来添加一个 Stage 并命名为 FirstStage

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/200731_130effca.png "在这里输入图片标题")

这个时候 Jenkins 提示我们至少要有一个 Step，我们点击右侧的 Add step 来添加一个 Step ，有很多定义好的 Step 供我们选择配置，这里我们选择 Print Message 来打印一个欢迎信息

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/200912_110a3ba9.png "在这里输入图片标题")

点击右上角的保存，会提示我们输入对这次改动的描述以及提交到哪个分支上，简单来说就是流水线编辑器会将这些改动映射到 Jenkinsfile 并且通过 Git 进行版本控制，这样的话整个编译构建的过程都能够通过 Jenkinsfile 管理起来，填写信息并提交

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/201057_b0fa4a36.png "在这里输入图片标题")

点击 Save & run 就会自动将文件提交到仓库并执行构建，查看我们的代码库，可以看到 Jenkinsfle 文件已经被提交

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/201252_23e6bdef.png "在这里输入图片标题")

回到 BlueOcean 查看执行状态，发现居然失败了

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/201335_396686c5.png "在这里输入图片标题")

看的出来，失败是因为服务器上并没有安装 Git，所以导致无法下载源码仓，我们去服务器上安装好 Git 并点击右上角的 rerun 图标重新执行

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/201555_7e1ca59b.png "在这里输入图片标题")

这次就继续执行了，BlueOcean 做的非常细节，失败的情况顶部是红色，运行中是蓝色，成功就是绿色，手动狗头

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/202439_e3152f92.png "在这里输入图片标题")

执行成功

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/202647_b407fecb.png "在这里输入图片标题")

我们在 Github 上查看最新的这次提交，发现已经有了一个绿色的钩

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/202805_8cc9e818.png "在这里输入图片标题")

我们点击右上角的铅笔，可以继续对流水线进行编辑

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0921/203105_f8c7ec41.png "在这里输入图片标题")

我们可以创建并行的任务，也可以创建多阶段的 Stage，并且有各种各样的 Step 供我们进行选择，通过对这些 Step 的组合和自定义，甚至于通过插件提供自定义的 Step，我们可以非常简单又方便的编辑我们的流水线，并且通过 Jenkinsfile 让它可以运行在任何 Jenkins 上。

### 技巧

为了能够实现每次推送自动进行构建，我们可以配置一个 Github 的 WebHook 进行自动触发构建，WebHook 的构成形式是:
```
http://{{JENKINS_URL}}/github-webhook
```
将这个地址添加到 Github 的 Webhook中去

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0922/134855_a85d768e.png "在这里输入图片标题")

这个时候我们来修改一下 Jenkinsfile 的内容，更改为 `Trigger me automatic!!!`并提交，这个时候我们来刷新 BlueOcean 界面，可以看到有一个构建自动被触发了

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0922/134716_12d00254.png "在这里输入图片标题")

### 总结

BlueOcean 提供了一种非常方便快捷的方式来自定义我们自己的流水线，就算是对于 Pipeline Syntax 不是特别熟悉的工程师，也可以通过流水线编辑器来快速的定义自己的流水线，并且可以与不同的版本控制系统进行集成，使得每一个提交每一个分支都可以通过这种方式来进行编译、构建、测试、质量检测等一些列可以提升代码质量和工程化效率的方法，接下来我将会从实际的应用来展开如何将 Jenkins 与实际的项目进行结合使用来提升研发效能。