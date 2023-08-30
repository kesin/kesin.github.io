---
layout: post
title: 问题排查神器 - Git Bisect 命令实战分享
date: 2021-08-29 08:13:00
excerpt: 前段时间 Git 发布正式版本 2.33.0 遇到一个客户端与服务端不兼容的一个问题，在排查问题的过程中又一次用到了 Git bisect 命令，解决问题的同时结合近期的一些认知，又刷新了自己对 bisect 命令的认识，bisect 可以结合一些场景发挥其妙用，借此分享下 bisect 命令以及 bisect 命令的一些其它使用方式的思考。
---

### 背景

Git 于 8 月 17 号发布了 2.33.0 正式版本，本次的发布内容可以参见：

> https://gitee.com/mirrors/git/blob/master/Documentation/RelNotes/2.33.0.txt

不幸的是，在 2.33.0 版本发布没多久，就有用户反馈 2.33.0 版本的 Git 在使用 SSH 通信协议的时候无法进行正常的 Clone/Fetch/Push 操作，具体的现象如下：

```
Cloning into 'xxxxxx'...
fetch-pack: unexpected disconnect while reading sideband packet
fatal: early EOF
fatal: fetch-pack: invalid index-pack output
```
看到这个提示第一时间就想到 Git 2.33.0 的更新日志里面好像是有关于 Fetch 和 Sideband 相关的更新，于是前往更新日志发现了如下的两个信息：
```
 * "git fetch" over protocol v2 left its side of the socket open after
   it finished speaking, which unnecessarily wasted the resource on
   the other side.
   (merge ae1a7eefff jk/fetch-pack-v2-half-close-early later to maint).

 * The side-band demultiplexer that is used to display progress output
   from the remote end did not clear the line properly when the end of
   line hits at a packet boundary, which has been corrected.

```
但是，为了进一步定位此问题的源头，方便精准的定位问题，所以使用了 Git Bisect 命令进行测试，找到具体的原因，才能够更好的去排查分析问题并加以解决。

### Git Bisect 介绍

> git bisect 官方文档：https://git-scm.com/docs/git-bisect

`git bisect` 命令的作用是使用二分查找法找到具体引起问题的 Commit。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0829/152811_ff72294f.png "在这里输入图片标题")

> 上图引自阮一峰的网络日志：http://www.ruanyifeng.com/blog/2018/12/git-bisect.html

简单来说就是我们给到 `bisect` 命令一个范围，它会自动的帮我们确认当前范围的中点，在这个中点上进行测试，并且告诉它这是一个好的提交（good commit）还是一个坏的提交（bad commit），进而来缩小查找范围，通过二分查找，我们就可以很快的定位到出问题的 Commit 以方便我们针对性的解决问题。

### 问题排查

由于我们已经确定了这个问题是 Git 2.33.0 版本引起的（因为测试了 Git 2.32.0没问题呀 :D），所以在使用 `bisect` 命令的时候，这个版本范围就是`v2.32.0` ~ `v2.33.0`，我们只要找到具体引入这个问题的提交，就能够更精准的定位问题，从而避免盲目的猜测和常识。

#### 进入 bisect 模式

我们下载 Git 最新源码，并且使用 `git bisect start` 命令来进入 bisect 模式

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0829/153820_f39ca49a.png "在这里输入图片标题")

`git bisect start` 命令指定了一个范围，并且 bisect 命令告诉我们，在这两个版本之间一共有304个提交，大概需要8步就可以定位到具体的 commit，这就是二分查找的好处。

#### 开始第一次测试

我们使用 `make` 命令进行 Git 源码编译构建，编译构建完成后，就可以使用 Git 提供的一个 wrapper 进行 Git 命令的调用，这里我们可以加上 `-j4` 参数来增加编译构建的速度。命令完成后，我们就可以使用 `./bin-wrappers/git` 来进行测试：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0829/154352_b1c0b896.png "在这里输入图片标题")

哦吼，还是不行，那么我们可以确定导致问题出现的 Commit 是出现在这次提交之后的提交里面。当然，这些不用我们自己来去记，使用 `git bisect bad` 命令标记即可：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0829/154725_22e7064c.png "在这里输入图片标题")

标记完成后 Git 就告诉我们，接下来还有大概7次的测试就可以定位到引发问题的 Commit，如果遇到的提交是可以通过的，那么需要使用 `git bisect good` 命令标记。

#### 自动化的 bisect

以上的步骤我们只需要重复的进行即可，但是很明显根本不需要人为的去跟进，作为一个新生代农民工，我们需要能够自动化可自动化的一切，所以一方面是为了省时间，另一方面是为了避免人为的失误，我们可以使用 `git bisect run` 命令来自动化的进行执行。
```shell
git bisect run my_script arguments
```
`bisect` 命令是以 `my_script` 脚本的返回值来确定当前的提交是好是坏，来看看官方的介绍：

> Note that the script (my_script in the above example) should exit with code 0 if the current source code is good/old, and exit with a code between 1 and 127 (inclusive), except 125, if the current source code is bad/new.

简单来说就是：
- 如果以 `0` 值退出，说明当前版本是好的
- 如果以 `1~124 126 127`退出，说明当前版本是坏的
- 如果以 `125` 退出，说明编译构建有问题

所以我们来根据这个规则写个脚本，但首先我们需要看看 Clone 失败的返回值是多少

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0829/155857_b7e21dc5.png "在这里输入图片标题")

OK, let's do it via Shell Scripts ~~~

```
#!/bin/zsh

# 每次将 Clone 的代码放到以当前 CommitID 为名称的目录，避免冲突
make -j4 && ./bin-wrappers/git clone git@gitee.com:/kesin/taskover.git `git log -n 1 --pretty=format:"%H"`
s=$?
if [ $s -eq 0 ]; then # 正常
  exit 0;
elif [ $s -eq 128 ]; then # 失败
  exit 1;
else # 其他情况，此处只是测试，建议针对不同的情况更加严谨点 XD
  exit 128;
fi
```
然后我们来执行这个脚本自动的进行二分查找定位：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0829/160440_ab237956.png "在这里输入图片标题")

这个时候 `bisect` 命令就会自动的执行编译构建和 Clone 过程，并且根据返回值自动的确定 Commit 的范围：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0829/160550_b1c13780.png "在这里输入图片标题")

这里是 `bisect run` 命令根据我们写的脚本自动的判定当前提交是一个坏提交，进而自动的进行下一步。当然，在二分查找的过程中也会遇到好的提交，`bisect` 命令将会根据我们提供的返回值自动的缩小范围：
![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0829/160636_afd4d3c1.png "在这里输入图片标题")

最终，`bisect` 将会为我们定位到第一个出现此问题的提交：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0829/161101_f54995e1.png "在这里输入图片标题")

** ae1a7eefffe60425e6bf6a2065e042ae051cfb6c is the first bad commit
 **

并且 `bisect` 命令也把这个导致此问题的提交的详细信息打印了出来，接下来我们就可以根据这个提交相关的改动来分析我们的问题。

### 分析并解决问题

上面我们使用 `bisect` 命令找到出问题的提交： https://gitee.com/mirrors/git/commit/ae1a7eefffe60425e6bf6a2065e042ae051cfb6c 

```
			/*
			 * this is the final request we'll make of the server;
			 * do a half-duplex shutdown to indicate that they can
			 * hang up as soon as the pack is sent.
			 */
			close(fd[1]);
			fd[1] = -1;
```

分析下提交的变更我们可以知道，这次变更主要是为了优化网络连接的占用，Git V2 via SSH 在传输的过程中，当客户端接收完数据后，还需要进行一系列的本地操作，这个操作过程已经不需要再维持跟服务端的链接了，所以需要在客户端发送完数据后就给服务端发送 FIN，进入半双工状态，等待服务端发送完数据后关闭连接即可，而无需经过漫长的本地操作后才关闭连接，从而避免了不必要的网络资源占用。

知道问题的原因之后，我们分析 Gitee 的 SSH 分发代理，发现我们的 SSH 代理在接收到客户端的 FIN 之后马上就会关闭这个 SSH 链接，进而导致上面的问题：客户端还没有接收完数据链接就提前断开了

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0829/155113_dcb52131.png "在这里输入图片标题")

解决的方式也很简单，在收到客户端的 FIN 之后不马上进行网络连接的关闭，而是等数据发送完之后才进行关闭。

### Git Bisect 使用的思考

现在的组织都在推崇提升研发效能，推崇 DevOps 文化及工具，是不是可以把 Bisect 这种逻辑用到整个流程中呢？

比如在自动化测试中，我们遇到一些测试不通过的 Case 的时候是直接失败的，虽然告知了具体的用例以及相关的输入输出，但是如果能够通过 Bisect 命令自动的找出第一个出现此问题的提交，那么对于组织无疑是有价值的：
- Case 不通过的同时直接给出了 Bad Case 的 Owner，精准通知，快速修复
- 在集中测试中，可以避免过度的通知，从而干扰到本来无关的人员
- 避免问题不明确，相互推诿，产生不良情绪，影响团队气氛
- ...

### 关于算法思想的思考

Git Bisect 所采用的二分查找思想我们耳熟能详，在 Git 源码中，同样采用二分查找算法的地方还有 Git Pack `idx` 文件的查找，通过精妙的扇区划分，加上二分查找算法快速定位到`object`的偏移量。除此之外，Git 中还采用了 SHA-1 Hash 算法、不同的 Diff 算法、大量的递归等。

```cpp
/* hash-lookup.c */
int bsearch_hash(const unsigned char *hash, const uint32_t *fanout_nbo,
		 const unsigned char *table, size_t stride, uint32_t *result)
{
	uint32_t hi, lo;

	hi = ntohl(fanout_nbo[*hash]);
	lo = ((*hash == 0x0) ? 0 : ntohl(fanout_nbo[*hash - 1]));

	while (lo < hi) {
		unsigned mi = lo + (hi - lo) / 2;
		int cmp = hashcmp(table + mi * stride, hash);

		if (!cmp) {
			if (result)
				*result = mi;
			return 1;
		}
		if (cmp > 0)
			hi = mi;
		else
			lo = mi + 1;
	}

	if (result)
		*result = lo;
	return 0;
}

```

但是在实际编码的过程中，有多少开发者能够拍着胸脯说：在编码过程中，我脑中有模型，心中有算法，能够以高效的方式、采用合理的逻辑去编写代码。

这个答案我想不言自明吧。

研究开源项目是一个很好的学习路径，能够从大量优秀的代码中学习到优秀的实践和思想，从研究学习到参与贡献，相比于这个过程，结果倒显得不那么重要了。

### 最后

善用工具，乐于思考，多多来 https://gitee.com/ 学习研究开源项目并作贡献。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0125/003521_d71b3ff1.png "在这里输入图片标题")

转载请保留出处：微信公众号「Zoker 随笔」（zokersay）