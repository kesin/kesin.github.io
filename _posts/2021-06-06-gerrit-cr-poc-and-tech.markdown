---
layout: post
title: 探究实现 Change Request 模式的三种方案
date: 2021-06-06 05:06:40
excerpt: 去年写过一篇文章《浅谈 Pull Request 与 Change Request 研发协作模式》，详细介绍了 PR Flow 以及 CR Flow 的区别，文末有说到 Gitee 应该提供推送分支自动创建评审的这项改进，刚好近期也听到很多关于平台支持 CR Flow 的声音，而且在不少客户内部也有类似需求，毕竟这个需求可以解放开发者的一部分精力，对于追求研发效能的当下，自然是一个不错的功能点，另外对于 Gerrit 转到 Gitee 的用户，也是一个平滑的过渡方案。所以写下这篇文章，探讨一下支持 CR Flow 平台的产品实现，并对 CR Flow 实现的底层原理做了一些实验性的工作，以供参阅。
---

### PullRequest & ChangeRequest 分别在说些啥？

 **PullRequest**  （也叫 MergeRequest）最初是由 Github 推出的一种开源协作模式，目前 Gitee、Gitlab等平台都支持这种协作模式，而且这种模式目前更多的用在了团队或者企业的内部代码评审层面，这种协作模式一般是由开发者 Fork 一个仓库，或者在仓库内新起一个分支如`zoker/feature-1`，无论是哪种方式，所有的大前提是：开发者没有目标分支的写权限，写权限是被严格控制的，必须经过 PullRequest 方式合入代码。

在相应的分支研发并自测完成后，通过 **Web 界面** 提交一个 PullRequest 请求将这些代码合入到目标分支，这个过程会有自动化测试、自动编译构建、代码评审、代码改进、代码合并等一系列操作，最终完成代码向目标分支的合入。

 **ChangeRequest**  是由 Gerrit 推出的一个概念，Gerrit 是为 AOSP（Android Open Source Project）写的，结合 [Repo](https://mirror.tuna.tsinghua.edu.cn/help/git-repo/) 工具用来管理庞大的安卓项目，多仓管理也是他的优势之一，但是更多的人把它视为代码评审神器，能够使每一个 Commit 都是可靠。这种模式同样需要严格管控目标分支的写权限，开发者可以在本地起一个分支进行开发。

与 PullRequest 不同的是，ChangeRequest 不需要开发者到 Web 上进行评审请求的提交，推送一个 Commit 到对应的分支之后，会自动创建一个评审单。而且另外一个比较大的区别就是，ChangeRequest 的评审是针对单 Commit 评审，而 PullRequest 针对的是一个或者多个 Commit 的集中评审。

> 详细说明参见：https://zoker.io/blog/talk-about-pullrequest-and-changerequest

通过以上的说明，我们大概可以明白了，原来这就是不同协作模式在工具上的区别而已。

现在人们追求研发效能的提升和改进，说到底就是消除一切可能的浪费，让开发者专注于业务的研发，其他的一些工作能自助化的绝不层层审批严卡，能自动化的绝不浪费开发者时间去自助操作。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1122/204531_9884d0bc.png "在这里输入图片标题")

然而 PR Flow 模式就需要开发者在提交完代码之后，手动的前往平台创建 PullRequest，对于很多已经规范化流程的企业来说，这就是一种浪费，因为：

1. 开发者在接收到卡片的时候已经指定了分支，只要他往这个分支推送代码，那肯定是解决分支对应卡片的问题，无需再进行 PullRequest 的描述的重复填写
2. 在提交卡片的时候，已经关联好了测试用例，PullRequest 无需再进行测试用例的补充，自动化的从卡片获取用例即可
3. 已经限制了开发者做提交的时候必须关联一个有效的卡片（如 fixed #JN7HE3），所以开发者在推送代码的时候，这些提交已经有了它们的身份，亦无须再手动创建 PullRequest
4. ...

CR Flow 恰巧就解决了这个问题，推送的时候可以自动创建评审，但是采用 PR Flow 的平台如 Github、Gitlab、Gitee 等是没有这个能力的，我做过类似在客户端定制钩子调用接口的实现，但是整体实现上并不顺畅，只有服务端统一处理，才能够考虑到各种情况。

所以这就是为什么很多做研发协作的 PR Flow 的平台都在补充 CR Flow 能力的原因。

### 如何实现推送自动创建评审？

相信一些了解 Git 的同学第一反应肯定就是 Git Hooks。没错，我在着手去做实验来实现相关逻辑的时候，第一个想到的就是 Update 钩子，但是 Update 钩子还是有一定局限性的。

不过去年10月份，Git 2.29 增加了一项新的能力 Proc-receive 钩子，这个能力是由阿里巴巴 Codeup 团队蒋鑫老师带领贡献的，有了这个钩子，在服务端能够自定义更新引用的工作，可以想象的空间就比较大了。

除此之外，还有一种方式就是 Gerrit 方式，我把这种方式称做帽子戏法，也就是自定义了服务端的 RPC 服务，在服务端做完一系列的处理之后，发给客户端我们想让他看到的数据即可。

所以，要实现推送自动创建评审，目前来看，有三种方式：

1. Gerrit 的帽子戏法
2. 自定义 Update 钩子
3. Git 新能力 Proc-receive 钩子

下面我们从产品和技术实现上一一展开阐述。

### Gerrit 的帽子戏法

Gerrit 实现 Change Request 的逻辑是是什么呢？Gerrit 规定了推送到指定的`ref`如：
```
git push origin HEAD:refs/for/master
```
就会进行基于最新的`master`进行`commit`的检索及评审的创建和更新。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/130031_2441f765.png "在这里输入图片标题")

如果`commit`还没有在`master`上面，那么就会生成一个新的 Change，对应的 Change 存储在`refs/changes`下面，如果没有的话则去`packed-refs`找。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/130053_7547b082.png "在这里输入图片标题")

- 其中`refs/changes/67/67/1`对应的就是这次提交的`commit`
- 而`refs/changes/67/67/meta`则记录此次 Change 的一些基本信息，里面有一个 Change-id 请记住它，这是 Gerrit 能够跟踪具体`commit`的关键

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/130245_4a40aaa7.png "在这里输入图片标题")

如果`commit`已经存在在`master`上，并且它被修改了，也就是`commit`还是那个`commit`，但是通过重写，`commit id`变了，那就会更新对应的 Change 生成一个新的 Patchset，同样存储在`refs/changes/67/67`下，只不过编号是`2`

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/130410_a04f32b7.png "在这里输入图片标题")

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/130418_7d59285b.png "在这里输入图片标题")

然而，Gerrit 怎么知道一个重写后的`commit`它的前身呢？答案就是上面的 Change-id ，Gerrit 通过用户配置的本地`commit-msg`钩子，将`msg`、`commit id`等信息 Hash 成一个 Change-id，这个 Change-id 就是贯穿整个 Change 生命周期的唯一标识符

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/130527_89439c0f.png "在这里输入图片标题")

那么问题来了，我推送到`refs/for/master`的数据，一不用加`-f`强推，二我又看不到拿不到`refs/for/master`的内容，感觉这个`refs/for/xxxx`就是一个`/dev/null`，往里面倒腾啥都行，关键是 Git 客户端还没报错，总是提示成功
```
* [new branch]      HEAD -> refs/for/xxx
```
这跟我们在用 Github、Gitee 平台的习惯不一样啊。没错，我刚开始接触 Gerrit 的时候，同样有这样的疑惑，猜想 Gerrit 应该是做了一些障眼法，服务端做了一些动作，告诉客户端：`好，你推送的这个分支成功了，你回去吧`。

当时去翻阅了 Gerrit 的源码：https://gitee.com/mirrors/Gerrit ，发现 Gerrit 是使用 Java 基于 JGit 自己实现的一套服务端的RPC逻辑，而不是像主流的平台那样使用 `receive-pack/upload-pack` 进行处理，所以才可以那么灵活的应对。

`java/com/google/gerrit/server/git/receive/ReceiveCommits.java`
```
/**
 * Receives change upload using the Git receive-pack protocol.
 *
 * <p>Conceptually, most use of Gerrit is a push of some commits to refs/for/BRANCH. However, the
 * receive-pack protocol that this is based on allows multiple ref updates to be processed at once.
 * So we have to be prepared to also handle normal pushes (refs/heads/BRANCH), and legacy pushes
 * (refs/changes/CHANGE). It is hard to split this class up further, because normal pushes can also
 * result in updates to reviews, through the autoclose mechanism.
 */
```

一般情况下，正常往一个分支推送应该是增量的，然后提示我们的信息类似于
```
803ed6b..5fd69aa  HEAD -> master
```
而推送到`refs/for/xxx` 的动作由 Gerrit 接管，并且在处理接受完数据之后（比如创建和更新 Change），就返回给客户端一个分支已经被创建的假象 
```
* [new branch]      HEAD -> refs/for/master
```
以此来保证客户端不会报错而疑惑，因为你请求的是更新/创建 `refs/for/xxx`，如果告诉客户端我创建了其他分支，那客户端就会报错，这受限于`report-status`能力，不过现在得益于阿里巴巴的贡献，已经新增了`report-status-v2`，这个能力就可以让客户端实现接受服务端实际的变更状态，比如更新一个 Change 后提示

```
* [new branch]      HEAD -> refs/changes/67/67/5
```

相信在不久后 Gerrit 应该会跟进。

> The 'report-status-v2' capability extends the protocol by adding new option
> lines in order to support reporting of reference rewritten by the
> 'proc-receive' hook. The 'proc-receive' hook may handle a command for a
> pseudo-reference which may create or update one or more references, and each
> reference may have different name, different new-oid, and different old-oid.

**那么，该如何自定义 `receive-pack` 来实现这种帽子戏法呢？**

我基于之前写的一个`http-smart-server`（https://gitee.com/kesin/go-git-protocols） 做了一个实验，在调用`receive-pack`接收完数据后，不把`receive-pack`的输出发给客户端，转而自定义`response`数据发给客户端，模拟一个正常推送`master`的`receive-pack`响应如下：
```
	pw := pktline.NewEncoder(w)
	// insert length msg
	// insert band protocol
	pw.Encode([]byte("unpack ok\n"))
	pw.Encode([]byte("ok refs/heads/master\n"))
	// flush pkt 0000
```
客户端在接收到这个响应的时候就能知道服务端正确的解压了包并且如期的完成了`refs/heads/master`的更新，那我们来搞点不一样的，比如我推送的是`master`分支，但是我让服务端返回`master1`被更新成功的信息：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/135717_e9c4ba88.png "在这里输入图片标题")

正如上面我们提到的，客户端并不认，因为这这与它的预期不符，转而报错了，我们再次修改代码：

```
	// callGiteeApi createPr(ref, new_oid, old_oid)
	// other custom operations
	pw := pktline.NewEncoder(w)
	// insert length msg
	// insert band protocol
	pw.Encode([]byte("unpack ok\n"))
	pw.Encode([]byte("ng refs/heads/master Your ref master is sleeping!\n"))
	// flush pkt 0000
```

我们在响应客户端之前，根据现有信息创建了 PullRequest 并且可以根据这次推送的 Context 信息做一些自定义的其他操作，然后把我们期望客户端看到的结果给它：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/140310_f239e62b.png "在这里输入图片标题")

通过这种帽子戏法的方式，我们就可以灵活的自定义服务端的处理，来实现我们需要的业务逻辑。

### 自定义 Update 钩子

一般情况下我们在日常使用到 Git 的过程中会涉及到三个服务端的钩子：

- pre-receive 钩子，用来做一些授权检查工作以及引用的批量更新操作，就算有多个`ref`一起更新，也只会执行一次
- update 钩子，每一个引用的更新都会触发，用来做版本自动化测试之类的工作
- post-receive 钩子，主要是用来更新引用后的一些工作，比如通知，触发其它服务等

所以在 Update 钩子做这个事情看起来比较合适，我写了一个 Update 钩子的 Proofs of Concept，用来演示，大概的逻辑：

1. 推送的时候检查这个分支是否设置了评审分支，如果设置了评审分支，那不允许直接推送，必须走PR
2. 根据推送的分支和用户进行判定，一个用户在一个评审分支只能有一个PR，类似{username}-{ref}
3. 自动创建完上述分支后，钩子通过api进行PR的创建
4. 如果已经存在pr，那么更新即可

```
package main

import (
	"fmt"
	"io"
	"os"
)

func main()  {
	if len(os.Args) != 4 {
		fmt.Println("Need 3 args: <ref> <oldrev> <newrev>")
		os.Exit(1)
	}
	ref := os.Args[1]
	oldrev := os.Args[2]
	newrev := os.Args[3]
	info := fmt.Sprintf("ref: %s, old: %s, new: %s", ref, oldrev, newrev)
	fmt.Println(info)

	if ref == "refs/heads/review" {
		reviewRefPath := fmt.Sprintf("%s/%s-zoker", os.Getenv("GIT_DIR"), ref)
		reviewRefExist := true
		refFile, err := os.Open(reviewRefPath)
		if err != nil {
			reviewRefExist = false
			refFile, err = os.Create(reviewRefPath)
		}
		// check non fast-forward between reviewRef and newrev and then exit 1 if needed
		io.WriteString(refFile, newrev) // if commit is fast forward
		// involve pr api to create pr and alert
		// createOrUpdateSucess := remote call result
		if reviewRefExist {
			fmt.Printf("Pushes have successfully update in PullRequest: https://gitee.com/xxx/xxx/pulls/xx \n")
		} else {
			fmt.Printf("You don't have permission push to %s \n", ref)
			fmt.Printf("We generate a new branch %s-zoker and automaticlly create a \n", ref)
			fmt.Printf("PullRequest for you: https://gitee.com/xxx/xxx/pulls/xx\n")
		}
		os.Exit(1)
	}
}
```
通过对`review`分支的检测，来做对应的处理：

如果当前用户在对应的目标分支还没有提交 PullRequest，那就创建并提示：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/144156_667d81a1.png "在这里输入图片标题")

如果当前用户在对应的目标分支已经提交了 PullRequest，那就更新：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/144215_06ecf42c.png "在这里输入图片标题")

上面默认是不冲突的情况，如果冲突的话，我们需要：

1. 如果`review-zoker`检测到与`newrev`冲突，可以不给更新
2. 如果`review`分支冲突的话，是无法继续推送的，这其实不太不合理，因为实际是要更新`review-zoker`

而且提示中夹在着错误信息，会对用户产生困扰，我在研究 ezOne 这个产品的时候，发现他们的做法应该是是围绕着 Update 钩子进行的，因为实际试用的情况与我的实验脚本结果是一样的：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/144650_edfdac3a.png "在这里输入图片标题")

只不过 ezOne 把这些提示信息加了颜色，着重展示给用户，但是使用 Update 钩子实现所产生冲突的问题并没有解决

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/144739_75d816ae.png "在这里输入图片标题")

此外 ezOne 应该也在返回给客户的响应上做了一些处理，或者结合使用了上面说的`proc-receive`钩子，因为拒绝`master`的推送并给出了提示`代码评审通过后方可合入`。整体来讲，ezOne 对于这块功能起了一个新的名字叫 DCR（Direct Change Request）。

**ezOne DCR 整体使用逻辑总结如下：**

1. 分支只要是保护分支，那么推送就会被拒绝，转而自动产生一个 DCR
2. 一个用户对一个目标分支可以有多个 DCR，如果`pushed..master`包含某个`pr..master`，那就更新，如果不包含那就新建
3. 会严格检查push与目标分支的冲突情况，如果强推也完全按照规则1进行的
4. 更新也只更新线性向前的 DCR，其他的情况并不会更新
5. 如果 Base 和目标分支冲突了，就直接冲突拒绝推送，没有找到此种情况更新 DCR 的方式

### Git 新能力 Proc-receive 钩子

上面对`proc-receive`钩子有过一些简单的介绍，归根结底就是`receive-pack`在更新引用的时候会把匹配到`receive.procReceiveRefs`的引用转交给新的钩子`proc-receive`进行处理，这个`proc-receive`钩子与`receive-pack`之间通过 PKT-LINE 格式的命令进行通讯。

两者之间的通信大体如下（我做了详细的注解，你应该可以看得明白）：

```
    # receive-pack 与 proc-receive 相互协商版本和能力
    S: PKT-LINE(version=1\0push-options atomic...)
    S: flush-pkt
    H: PKT-LINE(version=1\0push-options...)
    H: flush-pkt

    # receive-pack 在匹配到 receive.procReceiveRefs 的引用后，把这些引用的更新操作转交给 proc-receive 钩子进行处理
    S: PKT-LINE(<old-oid> <new-oid> <ref>)
    S: ... ...
    S: flush-pkt

    # pro-receive 钩子接收到这些信息，根据自己的逻辑进行对应的处理，新建引用，更新或者创建 PullRequest 可以安插到这里进行
    # 如果 proc-receive 钩子处理无误，则告诉 receive-pack 已经处理成功
    H: PKT-LINE(ok <ref>)

    # 如果 proc-receive 钩子处理失败，亦或创建或更新 PullRequest 失败，均可以自定义这些引用更新失败的信息
    H: PKT-LINE(ng <ref> <reason>)

    # 如果发现自己没法处理，甩锅给 receive-pack 继续处理即可
    H: PKT-LINE(ok <ref>)
    H: PKT-LINE(option fall-through)
```
> 如果你对 proc-receive 钩子实现的原理感兴趣，可以参见蒋老师的演讲文稿：
> https://git-repo.info/zh_cn/2020/03/agit-flow-and-git-repo/

基于上述对 `proc-receive` 钩子的理解，我使用 Go 简单的写了一个 `proc-receive` 钩子，代码如下：

```
package main

import (
	"os"
)

func main() {
	// init pkt reader and writer
	pd := NewPktDecoder(os.Stdin)
	pe := NewPktEncoder(os.Stdout)

	// get receive-pack version
	_ = pd.Decode(&[]byte{})
	_ = pd.Decode(&[]byte{})

	// send proc-receive version
	_ = pe.Encode([]byte("version=1"))
	_ = pe.Encode(nil)

	// scan pktline for matched receive.procReceiveRefs
	var c []byte
	_ = pd.Decode(&c) // c -> command a b refs/xxx/xxx
	_ = pd.Decode(&[]byte{})

	// ----------------------
	// do what you want, eg: generate ref, call api to create a pr on gitee...
	// newRef("refs/heads/zoker/xxx", ZERO_OID, c.new_oid)
	// createGiteePullRequest(new_ref, target_ref)
	// ----------------------

	//_ = pe.Encode([]byte("ok refs/heads/review"))
	_ = pe.Encode([]byte("ng refs/heads/review PullRequest created or updated, please........"))
	_ = pe.Encode(nil)
}
```

主要的功能就是与`receive-pack`进行通讯，然后告诉`receive-pack`分支`review`是否更新成功，我们可以在这里进行对应的逻辑处理，比如基于这次推送新建一个引用，或者为每个 Commit 都新建一个引用，基于这个引用去创建一个评审等

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/153125_5c083879.png "在这里输入图片标题")

阿里云的 Codeup 也就是基于 `proc-receive` 能力实现的 ChangeRequest 功能，不过整体的产品逻辑是参照 Gerrit 的，与 Gerrit 一样支持往 `refs/for/master` 推送生成 PullRequest 的，于此同时也支持 `refs/drafts/`、`refs/for-review/`等特殊的前缀，主要是在合入权限上的区别。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0606/154525_cab0abc1.png "在这里输入图片标题")

**Codeup 的 CR 功能使用总结如下：**

1. 同一个用户对一个目标分支只能创建一个CR
2. 推送到 `refs/for/master` 符合 Gerrit 的工作方式，不会报分之未更新的错
3. 更新、回退提交均可更新 CR，回退到目标分支之后，CR 提交就空了
4. 与源冲突的情况可以直接推送，不需要`-f`，直接覆盖原 CR
5. 严格按照 `HEA`D 与 `master` 的差异来更新 CR，不管是否冲突
6. 与目标分支冲突的情况可以直接推送，但是PR打不开了，一直转圈圈，有的关闭重开就好了，但有的关闭重开也不行
7. 如果有两个以上提交，CR 标题为 `merge from xxxxxx`，如果只有一个，标题就直接使用了这个提交的提交信息

此外，在使用腾讯工蜂的代码评审功能中，觉得工蜂的设计挺有意思，而且具体的实现也应该是使用了帽子戏法或者`proc-receive`钩子，工蜂提供了两个概念：合并请求、代码评审

- 合并请求：与常规的 PullRequest 功能逻辑没太大区别
- 代码评审：可以创建某一个或者多个提交的评审单，用于做代码评审

工蜂对分支提供了两种设置，一种是推送自动创建评审单，也就是无论你往目标分支上推送了什么，都会创建一个评审单用于代码评审，推送会成功更新到目标分支，不会造成中断；另外一种是推送自动创建评审单，但是会拒绝更新目标分支，需要通过评审才能继续推送新的提交。

**工蜂的 CR 功能使用总结如下：**

1. 保护分支不可强推，不可推送包含多个父提交的提交（可能我哪里没设置对）
2. 自动创建评审有两种选项，推送自动创建评审（仓库级），推送自动创建评审并且通过才能合并（保护分支）
3. 按照单次推送的行为来的（你推上来了什么，就用什么创建 CR，强推也一样，不存在更新的说法）
4. 保护分支的严格模式，推送自动创建了合并请求，看提示应该是修改了协议返回，自动创建了 `refs/for/zoker/master/1`，用时间戳作为合并请求的标题
5. 已经有合并请求的情况下，无法再次推送`master`，会提示 `remote: Code review must be approved before push`
6. 没找到更新这个自动创建的合并请求的方式（姿势不对还是设定如此？）
7. 关于冲突情况，如果跟目标分支冲突，直接提示冲突，拉取合并再去推送则提示
```
You are not allowed to push code to a protected branch on this project
```
删除 CR 也一样，移除保护分支也一样，尬住了，但是回退后重新产生新的冲突提交即可推送，然后强推到之前的版本，再把刚刚不能推送的合并再次推送，就可以了，估计是什么缓存逻辑被刷新了

#### 谈谈 Gitee 轻量级 Pull Request 的改进

Gitee 在去年推出了轻量级 PR 功能（https://gitee.com/help/articles/4291） ，旨在解决用户为开源项目提交贡献过程繁琐的问题。

从其本质上来看，轻量级 PR 要解决的问题其实与现在 Change Request 要解决的问题是一样的，都是为了减少用户的使用成本，使其专注代码层面的贡献。

轻量级 PR 功能目前只是在网页通过 WebIDE 进行提交，其实通过上面我们提到的一些技术，可以做到对于开源项目，直接 Clone 到本地进行修改，然后推送到目标仓库的某个分支即可，服务端可以针对推送进行验证，如果推送的为开源项目并且用户无授权，那么就自动生成或者更新 Pull Request 即可。

### 总结

目前，各个产品都在找寻一种合适的方式来覆盖用户对于自动创建评审的需求。但是我有一种很明显的感觉，就是每一个产品的实现逻辑，都包含了强烈的自身组织的研发协作方式，每种实现逻辑都可以覆盖一部分用户需求，但是并未能满足绝大多数，可能这也跟形形色色的研发协作模式有关。有时候想想，与其绞尽脑汁去想着如何在产品上满足不同用户的需求，倒不如自己定义一个标准，让用户来遵守和使用。

此外，对于 Change Request 功能的技术实现，`proc-receive`钩子的出现无疑降低了技术门槛，避免了对`receive-pack`业务逻辑的侵入。但无论使用哪种方式最终还是要从用户实际需求出发，抓住用户真正的痛点，才能通过技术真真正正的解决问题，而不是又创造了新的问题。
总结

最后，要感谢我的女朋友在进行这篇文章撰写时给我的鼓励和帮助，没有她的支持，我早就写完了。

**_对于 Change Request 这种协作模式，你有什么建议或者想法吗？_**