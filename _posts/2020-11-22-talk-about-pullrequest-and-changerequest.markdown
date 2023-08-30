---
layout: post
title: 浅谈 Pull Request 与 Change Request 研发协作模式
date: 2020-11-22 12:26:40
excerpt: 说起 PullRequest 相信大部分人都不会陌生，它是由 Github 推出的一种开源协作模式，由于 Gitlab 占据着企业内部私有部署的半壁江山，这种模式也更多的用在企业内部代码审核流程，也就是所谓的 CodeReview。其实还有很多企业和团队会选择 Gerrit 这个工具，Gerrit 提供的是 ChangeRequest 模式，这种模式更具有针对性，对代码审核的粒度也更细，近期有客户需求在 Gitee 上实现类似 ChangeRequest 的需求，所以针对两种模式做一个介绍，探讨两种模式的具体适用场景。
---

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204214_6414189d.png "在这里输入图片标题")

## 什么是 CodeReview

关于 CodeReview 是什么，[Gerrit 官方文档](https://gerrit-documentation.storage.googleapis.com/Documentation/3.2.3/intro-rockstar.html)给了一个非常形象的解释，一个软件或者 Feature 的生产过程，如同一首歌的问世前一样，需要歌手反复录制不同的片段达到最佳的效果，要保证每一个片段都令人满意，并且要把所有的片段融合起来，最终目的就是把最好的版本提供给大家品鉴。Queen 乐队有一首《波西米亚狂想曲》，用了3周进行录制，其中有一些片段甚至重复录制了180多次，最终才有了这首经典。

同样的情况也在软件工程领域每天上演着，开发者们根据同事的反馈，一遍又一遍的修改、优化自己的代码，最终产出一个高质量版本的产品交付到用户手中，这就是 CodeReview（代码评审）所做的事情。

一件理论或者实践的推动，总是要有相应的价值产生。那么 CodeReview 能够给我们带来什么呢，来看看 Gerrit 的总结：

1. Code reviews mean that every change effectively has shared authorship
2. Developers share knowledge in two directions: Reviewers learn from the patch author how the new code they will have to maintain works, and the patch author learns from reviewers about best practices used in the project.
3. Code review encourages more people to read the code contained in a given change. As a result, there are more opportunities to find bugs and suggest improvements.
4. The more people who read the code, the more bugs can be identified. Since code review occurs before code is submitted, bugs are squashed during the earliest stage of the software development lifecycle.
5. The review process provides a mechanism to enforce team and company policies. For example, all tests shall pass on all platforms or at least two people shall sign off on code in production.

> 摘自： https://gerrit-documentation.storage.googleapis.com/Documentation/3.2.3/intro-rockstar.html#review

简单翻译一下：

1. 让每一个提交（变更、改动）都有多个 Owner（好处就是出了问题不会有单点故障，任何一个 Owner 都可以处理）
2. 双向的知识传递：评审人可以从提交者那里看到新的代码如何运作的，提交者可以从评审者那里学会项目中的一些最佳实践
3. 代码评审能够提供一个正向的机制，让更多的人参与到评审中来，那么就会有更多的 Bug 或者提升被发现
4. 越多的人评审了代码，Bug 被发现的几率越大，Bug 可以在整个软件开发生命周期的前期就被处理掉
5. 这种评审可以助力团队及公司制度的完善，比如可以规定投产的代码必须有两个人以上的签名才可以

## PullRequest & ChangeRequest 简介

 **PullRequest**  最初是由 Github 推出的一种开源协作模式，目前 Gitee、Gitlab等平台都支持这种协作模式，而且这种模式目前更多的用在了团队或者企业的内部代码评审层面，这种协作模式一般是由开发者 Fork 一个仓库，或者在仓库内新起一个分支如`zoker/feature-1`，无论是哪种方式，所有的大前提是：开发者没有目标分支的写权限，写权限是被严格控制的，必须经过 PullRequest 方式合入代码。在相应的分支研发并自测完成后，通过 **Web 界面** 提交一个 PullRequest 请求将这些代码合入到目标分支，这个过程会有自动化测试、自动编译构建、代码评审、代码改进、代码合并等一系列操作，最终完成代码向目标分支的合入。

 **ChangeRequest**  是由 Gerrit 推出的一个概念，Gerrit 是为 AOSP（Android Open Source Project）写的，结合 [Repo](https://mirror.tuna.tsinghua.edu.cn/help/git-repo/) 工具用来管理庞大的安卓项目，多仓管理也是他的优势之一，但是更多的人把它视为代码评审神器，能够使每一个 Commit 都是可靠。这种模式同样需要严格管控目标分支的写权限，开发者可以在本地起一个分支进行开发，与 PullRequest 不同的是，ChangeRequest 不需要开发者到 Web 上进行评审请求的提交，推送一个 Commit 到对应的分支之后，会自动创建一个评审单。而且另外一个比较大的区别就是，ChangeRequest 的评审是针对单 Commit 评审，而 PullRequest 针对的是一个或者多个 Commit 的集中评审。

下面就以 Gitee 和 Gerrit 为例，从几个方面简单介绍下两者在日常研发协作上的区别

## Diff_1：使用

从使用上来讲，PullRequest 以及 ChangeRequest 的最大的差别在以下几点：

#### 1. 评审的创建

PullRequest 需要我们的研发人员在做完一项研发工作后，在推送相应的特性分支到 Gitee 后，需要在 Gitee 上进行评审单的创建；而 ChangeRequest 则不需要，因为在推送后，Gerrit 会自动创建一个评审，也就是一个 ChangeRequest。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204256_9f53df91.png "在这里输入图片标题")

PullRequest 由于是开发者直接提交的，所以可以在提交 PullRequest 的时候就写好各种描述信息；而 ChangeRequest 仍需要前往 Gerrit 的 Web 界面进行重新完善

Gitee 创建 PullRequest 界面

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204322_894cc94b.png "在这里输入图片标题")

#### 2. 评审的内容

PullRequest 一般都是包含一个以上的 Commit 的评审，是一个 Commit 集合。当开发者接受到修改意见后，一般都是直接基于当前的提交继续进行提交，来处理代码的改进，Gitee 会自动更新 PullRequest 来更新这些改动，在一个 PullRequest 的生命周期中，可能会产生多个提交。因为在最终可交付的时候，整个 PullRequest 可能存在很多提交，这些提交可能是代码改进，也可能是冲突的合并，不过在合并的时候可以有多种形式的 Merge 方式，可以选择是将历史合入还是重新整合为一个 Commit 进行提交。

Gitee 多种合并方式的选择：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204420_8b35b301.png "在这里输入图片标题")

而对于 ChangeRequest 来说，则是针对于单 Commit 的评审，每个 Change 都只有一个提交。对代码的改进一般都是修改当前 Commit，也就是`git commit --amend`，Gerrit 接受到新的提交后，会自动更新这个 Change，并且会保留之前的提交，作为本 Change 的一个 [Patchset](https://gerrit-documentation.storage.googleapis.com/Documentation/3.2.3/concept-patch-sets.html)。同样，在一个 ChangeRequest 生命周期中，可能会有多个 Patchset 产生，不过在最终合并的时候合入的是经过多次修改后最优质的那个 Patchset。

Gerrit 的一个 Change 包含多个 Patchset：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204433_be8b77dc.png "在这里输入图片标题")

那肯定会有人有疑问，我每次推送到同一个 ChangeRequest 的 Commit，由于修改了内容或者描述，CommitID 早就变了，Gerrit 如何辨别区分是同一个修改？这就是 Gerrit 设计上优秀的地方，虽然 CommitID 变了，但是可以通过 Git 的钩子插入一个 [ChangeID](https://gerrit-documentation.storage.googleapis.com/Documentation/3.2.3/user-changeid.html)，这个 ChangeID 就是识别同一个 Commit 的重要武器，详细可以参阅官方文档，这里不再赘述。

以上两个是在使用上最主要的差异，Gitee 提供的 PullRequest 功能对功能需求拆分度要求不高，因为我们可以在同一个 PullRequest 里面做很多事（这是不推荐的），但是对于 Gerrit 的 ChangeRequest 来说，需要对需求拆分的相对独立，需求粒度一定要小，相互之间不可以相互影响，否则对于按照 Commit 进行评审的模式来说，各种冲突依赖就乱套了。

## Diff_2：管理
从配置管理员的角度来看，Gitee 和 Gerrit 的配置管理差别还是比较大的，尤其是权限管理。

Gitee 对于权限的管理主要是从开发人员角色、分支权限等角度进行配置，相对来说比较简答，配置项清晰明了，可以通过简单的配置即可实现 PullRequest 协作流程的基础功能。从使用成本上来讲是非常高效的，各个权限隔离也比较清晰，就算是一个入门的配置管理员也能够快速理解并完成配置。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204444_dbe01d8a.png "在这里输入图片标题")

Gerrit 的配置相对来说就比较复杂了，但是更加灵活，各个配置项均可灵活配置，但是灵活带来的就是使用成本的提升，使用成本相对 Gitee 比较高。不过好在提供了全局的继承功能，能够在已经有配置模版的情况下快速覆盖，Gerrit 中可以自建用户组，并且可以对不同项目的不同用户组进行灵活的权限配置。坏处就是如果一不小心动到一个并不了解的配置项，可能就会造成代码权限的泄漏，所以对配置管理员的要求更高。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204505_7eea8933.png "在这里输入图片标题")

## Diff_3：多仓管理

在一些大型的工程如安卓项目、鸿蒙OS等，将所有的代码放到一个代码库显然是不可能的事情，一方面是体积过大，无法高效管理，严重降低开发者的效率；另外一个重要的方面是权限管理，每个开发者都拥有一整个系统的代码显然是不合适的。

那么，该如何解决这种问题呢，Git 给出了他自己的两种解决方案：

- Git Submodule 
- Git Subtree

Git Submodule 是目前用的比较多的一种解决方案，主要的功能就是将其他仓库的某个版本作为当前仓库的一个目录，挂在当前仓库下，所有的配置信息都存放在当前仓库的`.gitmodules`文件下，包含了子仓库的信息以及映射的目录

```
[submodule "lib"]
	path = lib
	url = https://gitee.com/xxxxx/lib.git
```
这样做的好处是`lib`目录可以单独进行权限的管理，不至于权限过于发散。Git Subtree 的实现类似，它是 Git 1.5.2 之后官方新增的一个功能，旨在替代 Git Submodule 管理共用仓库，两种方式都可以达到多仓管理的功能，只不过 Submodule 是引用，而 Subtree 是拷贝，具体的使用可以参见 [Git - - subtree与submodule](https://www.cnblogs.com/anliven/p/13681894.html)。Submodule 以及 Subtree 在主流的代码托管平台都支持，Gitee 也一样，毕竟这是 Git 的 Native Feature。

那么，为什么已经有了 Submodule 了，谷歌团队还要搞一个 Gerrit 出来呢？主要还是管理上的不方便，开发上的效率低下，所有才有了 Gerrit + Repo 这套工具。

如上所述，Repo 工具是谷歌开发的用于管理 Android 版本库的一个工具。Repo 并不是用来取代 Git的，它是使用 Python 对 Git 的一层封装，简化了对多个 Git 版本库的管理方式。Repo 主要是结合着 Gerrit 来使用，它以一个`manifest.yml`为中心，通过对项目结构化的描述，达到与 Submodule 同样的效果，但是比 Submodule 更加灵活和方便。

```
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote  name="gitee"
           fetch="git@gitee.com:{namespace}"     
           autodotgit="true" /> <!--fetch=".." 代表使用 repo init -u 指定的相对路径 也可用完整路径，example:https://gitee.com/MarineJ/manifest_example/blob/master/default.xml-->
  <default revision="master"
           remote="gitee" /><!--revision为默认的拉取分支，后续提pr也以revision为默认目标分支-->

  <project path="lib" name="lib" />  <!--git@gitee.com:{namespace}/{name}.git  name项与clone的url相关-->
  <project path="src" name="src" /> 

</manifest>
```
为什么说 Repo 主要是结合着 Gerrit 来使用呢？因为 Gerrit 的特性就是推送新的提交可以自动创建评审，那么在一个大型工程的研发流程下，一个功能经常性的需要跨仓库进行开发，开发完成后需要提交评审单进行评审。如果一个功能涉及到了三个仓库，那么在进行了对应的开发后，只需要执行 Repo 提供的相关命令，即可在 Gerrit 上产生映射在三个仓库的三个 ChangeRequest，并且还可以在 Repo 命令追加评审人、话题等一系列属性。

当然，Repo 工具如果作为批量仓库提交工具，也是可以在 Gitee 上使用的，但是推送完成之后呢？需要开发者手动创建三个 PullRequest！这无疑带来了非常大的开发成本，开发者也会因此而抱怨。所以 Gerrit + Repo 这套工具其实就是为了解决大型工程的项目管理及代码评审而设计的。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204531_9884d0bc.png "在这里输入图片标题")

不过，在今年年中的时候，我们 Gitee 团队对 Repo 工具进行了 Fork Flow 的支持，让 Repo 工具可以在推送之后，自动在相关的仓库里面创建 PR，免去了复杂的创建流程，相关使用和实现可以参见 [oschina/repo](https://gitee.com/oschina/repo)

## Diff_4：持续构建

前面我们介绍了可以使用 Submodule 或者 Submodule 来统一管理多仓的场景，在这种模式下，持续构建变得也容易了，毕竟所有的代码都是主仓库拥有的。这里以 Jenkins 为例，我们只需要配置 [Gitee-Jenkins-Plugin](https://gitee.com/oschina/Gitee-Jenkins-Plugin) 插件即可， [Gitee-Jenkins-Plugin](https://gitee.com/oschina/Gitee-Jenkins-Plugin) 是 Gitee 团队基于 [GitLab Plugin](https://github.com/jenkinsci/gitlab-plugin) 二开的一个 Jenkins 插件，包含但不限于以下功能：
- 推送代码到 Gitee 时，由配置的 WebHook 触发 Jenkins 任务构建
- 提交 Pull Request 到码云项目时，由配置的 WebHook 触发 Jenkins 任务构建，支持PR动作：新建，更新，接受，关闭，审查通过，测试通过
- 按分支名过滤触发器。
- 构建后操作可配置 PR 触发的构建结果评论到码云对应的PR中。
- 构建后操作可配置 PR 触发的构建成功后可自动合并对应PR。
- ...
更多的可以参见项目的 [Readme](https://gitee.com/oschina/Gitee-Jenkins-Plugin#%E7%9B%AE%E5%BD%95)

可是，如果使用 Repo 工具在 Gitee 进行多仓管理呢？所有的仓库都是独立的，我们该如何知道哪些仓库变更了呢？在 [oschina/repo](https://gitee.com/oschina/repo) 的实现中，我们也遇到了这个问题，当时与客户沟通提了这么个解决方案：用户在提交代码后，Repo 工具会自动创建 PullRequest，与此同时，开发者将仓库以及对应的 PullRequest 信息提交到`manifest`仓库的一个 Issue，通过 Issue 的 Webhooks 触发流水线，流水线获取信息后进行对应的编译构建操作。这么做虽然可以解决问题，但是太不优雅了，对开发者非常不友好。

那么，Gerrit 如何处理这种情况呢？

在 Gerrit 的一个 Change 中，可以创建 [Topic](https://gerrit-documentation.storage.googleapis.com/Documentation/3.2.3/concept-changes.html#topic) 将多个 Change 关联起来，这样就可以使用 Jenkins 上面的两个插件： [Jenkins Repo](https://plugins.jenkins.io/repo/) 和 [Gerrit Trigger](https://plugins.jenkins.io/gerrit-trigger/)

- Jenkins Repo: This plugin adds Repo as an SCM provider in Jenkins.
- Gerrit Trigger: This plugin integrates Jenkins to Gerrit code review for triggering builds when a "patch set" is created.

可以利用 Topic 这个特性，结合这两个插件，通过配置，在有新的 Patchset 提交的时候，通过 Topic 的关联，来针对性的拉取有更新的 Change 进行编译构建。这里需要注意的是，Repo 工具每次的提交可能是多个仓库同时进行推送的，因为开发者在本地的行为是改动多个依赖仓，本地验证通过后一并提交的。

使用 Repo 作为仓库源的配置：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204544_5ef39a20.png "在这里输入图片标题")

使用 Gerrit Trigger 进行灵活的配置：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204554_7de4eaeb.png "在这里输入图片标题")

## 关于融合的一点思考

话说天下大势，分久必合，合久必分，道理放到哪里都一样。使用 Gerrit 就必须重视配置管理，对于一些使用体验上就没有那么便捷，前期的使用培训成本较高，Review 过程相对隔离，对于一直使用 PullRequest 模式的开发者并不是特别友好。而 Gitee 能够让我们快速构建起一个标准的研发协作流程，但在使用 Gitee 提供的各种方便功能，减少我们配置的复杂度的同时，对于多仓管理及评审的场景，Gitee 并不是强项，但是 Gitee 也在进步：

- 提供了按照 Commit 进行评审的交互，部分客户还做了 Commit 检视，能够召集团队对所有 Commit 一个一个进行评审并切可以打标记
- 提供了 Repo 的 Fork Flow 模式

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204609_e87bb9a4.png "在这里输入图片标题")

Gitee（其它平台也一样）可以做的事情也还有很多：

1. 提供代码审核不同接口的门禁接口，更加方便与不同的自动化测试及编译构建对接，用结果变更状态
2. Repo Fork Flow 可以改造为自动提交 Issue 变更单，结合流水线达到 Gerrit 的 Topic 的效果
3. 提供推送分支自动创建评审的功能（其实 Gitee 已经有轻量级PR了，只是没对本地推送自动创建做支持）
4. PullRequest 的 Commit 支持类似于 Gerrit 的逐个审查，进一步协助团队提升代码质量
5. ...

Gerrit 也有可改进的点：

1. 整合相关联的 Commit 到一个 ChangeSet 集合视图，提供类似于 PullRequest 的评审体验
2. 提供更多的组织架构层级显示的功能，方便管理和整合权限
3. 提供统计报表等度量工具，协助团队管理及团队内部竞争氛围的提升
4. ...

最后，附一张来自 AOSP 的关于 Gerrit 的《Life of a patch》供大家参考学习

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204627_d81b333c.png "在这里输入图片标题")

（END）