---
layout: post
title: 一起来玩 Jenkins:（4）为 Blogine 博客系统配置持续发布的能力
date: 2020-10-27 17:35:09
excerpt: 前面我们介绍了 Jenkins 相关的流水线以及可视化编辑流水线的 BlueOcean 插件，接下来我们主要通过一些实际的应用，来介绍下 Jenkins 在实际工作中的应用，同时也能够避免枯燥乏味的功能介绍，直接切入应用，这样来的比较有成就感。近期也是因为公司工作调整，突然间多了很多事，所以停更了一个月，晚上睡觉想起倍感羞愧，不敢睡觉，于是赶紧补上此篇博客来弥补内心的空缺 ;D
---

### 前言

2016 年的时候想要做一款属于自己的开源项目，思来想去没有什么好的想法，刚好 Ruby on Rails 并没有一个特别流行的博客系统，于是想着自己来造个轮子，项目已经上线 https://zoker.io ，整个项目开发比较简单，在 `master` 分支上进行开发，定期打`Tag`进行发版，这里通过介绍如果`master`分支有更新，该如何自动更新线上的博客系统，实际上合适的操作应该是依据版本进行发布的，这样比较合理且安全，因为你打版本肯定是觉得这个版本足够稳定才进行的操作，这里为了介绍功能，就不搞那么规范化了。看完此篇，你将会通过 Jenkins 持续发布一款 Web 应用，避免提交完代码还要手动登陆服务器进行更新。

### 流程说明

刚开始玩 Jenkins 的时候是把自己的博客服务器添加为了 Agent，虽然可以实现功能，但是实际上 Agent 是为了分发构建压力而设定的场景，用于生产的部署显然不合适，后来找到了一款插件`Publish Over SSH`，可以很方便的实现构建并发布到生产服务器。

整个流程无非就是：创建一个流水线 -> 代码有改动进行编译构建（这里的项目是 Ruby，脚本语言，所以可以忽略构建这部）-> 通过`Publish Over SSH`远程执行部署命令 -> 结束

### 插件安装

我们打开 Jenkins 插件中心，搜索 `Publish Over SSH` 并安装

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1027/234817_b1c62bef.png "在这里输入图片标题")

### 配置`Publish Over SSH`插件

安装完成之后，进入 Jenkins 配置管理中心，配置`Publish Over SSH`插件，主要是配置 SSHKeys 以及需要投产的机器，首先确保你的 Jenkins 主机能够通过 SSHKey 免密链接你的生产机器，然后填写相关信息即可

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/001427_2b9348e8.png "在这里输入图片标题")

需要注意的是，我在测试配置的过程中遇到了下面的错误，报私钥无效

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/000836_b3e27047.png "在这里输入图片标题")

经过查证发现是生成私钥的版本过高，Jenkins 无法识别，我们只需要指定使用旧版的私钥即可
```
ssh-keygen -m PEM -t rsa
// -m 参数指定密钥的格式，PEM是rsa之前使用的旧格式
```
### 配置部署流水线

我们进入新建界面，创建一个自由风格的项目

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/001833_2751b860.png "在这里输入图片标题")

配置源码库，这里是 Blogine 仓库开源地址，并配置对`master`分支构建

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/002157_1142a3bd.png "在这里输入图片标题")

配置触发方式，这里我们选择结合 WebHooks 进行触发构建

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/002522_53a608d1.png "在这里输入图片标题")

将得到的地址配置在 Gitee 仓库的 WebHooks 内

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/002614_7b0ed839.png "在这里输入图片标题")

接下来配置构建步骤，一般情况下，对于编译型语言，需要在本地构建好制品包，然后分发到生产机器进行部署的；但是对于脚本语言，直接在服务端部署即可，不过一些前端资源的打包可以先在本地做了，这里我们直接在构建这步进行服务端的部署操作

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/010205_23d02265.png "在这里输入图片标题")

点击保存，我们先手动触发一下构建，然后进入构建查看日志

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/010404_e33248dc.png "在这里输入图片标题")

可以看到命令都已经被正常执行了，登陆到服务器查看进程时间

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/010454_a5ce08a3.png "在这里输入图片标题")

可以看到已经正常启动

以上是我们自行手动触发的，接下来我们来进行一个真实的操作，实际的投产一个改进。

### 实际应用

目前 https://zoker.io 默认展示8条博客，我觉得太少了，无法展示出近期的关注的点，主要是人比较懒，写的比较少，所以就想展示的多一些，我们计划来改成默认展示16篇

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/010740_86ad259a.png "在这里输入图片标题")

查看代码 https://gitee.com/kesin/blogine/blob/master/app/controllers/posts_controller.rb ，发现在 `index` 这个`action`中

```
  def index
    @posts = Post.releases
    @posts = @posts.unconcealed if current_user.blank?
    @posts = @posts.sorted_by_created.includes(:tags, :column).page(params[:page]).per(8)
  end
```

我们把`.per(8)`改为`.per(16)`，每页展示16篇，然后提交。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/011128_ace38183.png "在这里输入图片标题")

提交完成后我们进入 Jenkins 就可以看到一个新的构建已经被触发了

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/012638_036fea53.png "在这里输入图片标题")

查看日志可以看到已经拉去代码并重启了

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/012858_6338c941.png "在这里输入图片标题")

我们再次前往博客查看首页展示博客数量，已经变成16篇了

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1028/013009_6469a658.png "在这里输入图片标题")

### 总结

经过了一系列的配置，我们终于可以解放双手，在更新了博客的一些功能之后，可以自动更新的服务器，不需要我们自己再登陆服务器并进行更新操作了。

当然，这种操作在企业级应用研发背景下显然是不合适的，中间还要经过代码质量检查、自动化测试、构建物审计、性能测试等等一系列可以提升产品质量的步骤，这些都可以融入到整个研发流程当中去，能够极大的简化我们的人工成本，使我们能够快速迭代，快速反馈，快速对产品进行提升，助力产品在市场中赢得头筹。