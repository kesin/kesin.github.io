---
layout: post
title: 聊聊 Git 的三种传输协议及实现
date: 2021-01-24 15:24:43
excerpt: 众所周知，Git 是当前最流行的分布式版本控制系统，近两年由于 DevOps 的迅速发展，一切皆代码正在成为标准实践。而这一切，都需要有一个配置管理中心统一管控，Git 毫无疑问的成为了这个领域的宠儿。日常开发工作中，我们经常用不同方式去下载和上传代码到 Gitee，那么这背后是如何实现的呢，让我们一起来聊聊 Git 不同的传输协议以及具体的实现。
---

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0121/125142_e17beffe.png "在这里输入图片标题")

前段时间在 InfoQ 公开课分享了 [《Gitee 架构演进之路》](https://live.infoq.cn/room/602)主题（回放地址：https://live.infoq.cn/room/602），中间有简单介绍了集中 Git 传输协议，并展开讲解了一下 HTTP 协议的传输机制，分享后不少同学通过公众号反馈有没有更详细的实战介绍，除了推荐一些博客、官方文档，好像也没有更好的资料，于是想着写写文章来聊聊自己的理解。

Git 在[官方文档](https://git-scm.com/book/zh/v2/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E5%8D%8F%E8%AE%AE)介绍了四种传输协议，并且对比了它们之间的优劣，这里就不再赘述了，感兴趣的可以翻阅上面公开课的PPT或者直接查看官方文档。另外在[源码设计文档](https://github.com/git/git/blob/master/Documentation/technical/)非常详细的介绍了HTTP、SSH 及 Git 三种传输协议的定义和规范，后续所有的介绍均围绕官方文档进行，与官方文档不同的是本系列文章会通过具体的实战，使用 Go 语言来实现相关协议 Git 服务端，以此来加深对相关传输协议的理解。
> Git 协议介绍：https://git-scm.com/book/zh/v2/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E5%8D%8F%E8%AE%AE
> 源码设计文档：https://github.com/git/git/blob/master/Documentation/technical/

## HTTP 传输协议

Git HTTP 协议主要分为两种，一种是哑协议（Dump），另外一种是智能协议（Smart），也是目前各个提供 Git 托管服务普遍所采用的协议。

> 官方文档：https://github.com/git/git/blob/master/Documentation/technical/http-protocol.txt

### HTTP 哑协议（Dump Protocol）

在 Git 1.6.6 版本之前是只提供哑协议的，哑协议只需要一个标准的 HTTP 静态文件服务，这个服务只需要能够提供文件的下载即可，Git 客户端会自动进行文件的遍历和拉取。

无论是哑协议还是智能协议，Git 在使用 HTTP 协议进行 Fetch 操作的时候，总是要先获取`info/refs`文件，这个文件是在裸仓库的目录下的，如果你已经有一个通过 Git 拉取的仓库，这个文件就在仓库根目录的`.git/info/refs`。不过这个文件一般情况下是没有的，它需要你在相应的目录执行`git update-server-info`进行生成：

```
➜  .git git:(master) cat info/refs
21f45f60fa582d085497fb2d3bb50163e59891ee	refs/heads/history
ef8021acf4c29eb35e3084b7dc543c173d67ad2a	refs/heads/master
```
文件内容主要是服务端上每个引用的版本，客户端拿到这些引用之后，就可以跟本地的引用进行对比，对于缺失的对象文件，则通过 HTTP 的方式进行下载。
> Tips1: 关于 Git 存储格式请参见：https://github.com/git/git/blob/master/Documentation/gitrepository-layout.txt，后续文章会展开介绍

> Tips2: 如果有更新都要手动执行`update-server-info`？答案是 No，可以配置 Git 服务端的`post-receive`钩子自动执行更新

所以，一次通过哑协议 Clone 的过程如下：（U：用户 C：客户端  S：服务端）

1. U：git clone https://gitee.com/kesin/taskover.git
2. C：GET https://gitee.com/kesin/taskover.git/info/refs
3. S：Response with `taskover.git/info/refs`
4. C：GET https://gitee.com/kesin/taskover.git/HEAD （默认分支）
5. S：Response with `taskover.git/HEAD`
6. C：Get https://gitee.com/kesin/taskover.git/objects/ef/8021acf4c29eb35e3084b7dc543c173d67ad2a 开始遍历对象，找出那些本地没有的，去服务端获取，如果服务端无法直接获取，则从 Pack 文件中进行抓取，直到全部拿到
7. C：根据 HEAD 中的默认分支执行 `checkout` 操作检出到本地

> 上面的那些地址是为了演示用，实际 Gitee 仅支持智能协议而不支持哑协议，毕竟对于一个公有云服务是不安全的。关于对象如何遍历这里也不再展开，后续文章会介绍

哑协议的实现非常简单，通过`nginx`即可简单实现，只需要配置一个静态文件服务器，然后将 Git 仓库以单目录的形式放上去即可；也可以使用 Go 快速实现一个简单的 Git HTTP Dump Server：

```
// From: https://gitee.com/kesin/go-git-protocols/blob/master/http-dumb/git-http-dumb.go

// Usage: ./http-dumb -repo=/xxx/xxx/xxx/ -port=8881

func main() {
	repo := flag.String("repo", "/Users/zoker/Tmp/repositories", "Specify a repositories root dir.")
	port := flag.String("port", "8881", "Specify a port to start process.")
	flag.Parse()

	http.Handle("/", http.FileServer(http.Dir(*repo)))
	fmt.Printf("Dumb http server start at port %s on dir %s \n", *port, *repo)
	_ = http.ListenAndServe(fmt.Sprintf(":%s", *port), nil)
}
```

### HTTP 智能协议（Smart Protocol）

HTTP 智能协议与哑协议最大的区别在于：哑协议在获取想要的数据的时候要自行指定文件资源的网络地址，并且通过多次的下载操作来达到目的；而智能协议的主动权则掌握在服务端，服务端提供的`info/refs`可以动态更新，并且可以通过客户端传来的参数，决定本次交互客户端所需要的最小对象集，并打包压缩发给客户端，客户端会进行解压来拿到自己想要的数据，整个交互过程如下：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0121/154819_3bd0eef6.png "在这里输入图片标题")

通过监听对应端口，我们可以看到整个过程客户端发送了两次请求：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0122/182635_83f6b4dd.png "在这里输入图片标题")

1. 引用发现 `GET https://gitee.com/kesin/taskover/info/refs?service=git-{upload|receive}-pack`
2. 数据传输 `POST https://gitee.com/kesin/taskover/git-{upload|receive}-pack`

Git HTTP 协议要求无论是下载操作还是上传操作，都必须先执行引用发现，也就是需要知道服务端的各个引用的版本信息，这样的话才能让服务端或者客户端知道两方之间的差异以及需要什么样的数据。

#### 1. 引用发现

与哑协议不同的是，智能协议的的服务端是动态服务器，能够根据期望来提供相关的引用信息，你可以根据自己的业务需求来决定想让客户端知道什么样的信息，通过抓包我们可以看到客户端请求的数据以及 Gitee 服务端返回的引用信息格式

```shell
# 请求体
GET http://git.oschina.net/kesin/getingblog.git/info/refs?service=git-upload-pack HTTP/1.1
Host: git.oschina.net
User-Agent: git/2.24.3 (Apple Git-128)
Accept-Encoding: deflate, gzip
Proxy-Connection: Keep-Alive
Pragma: no-cache

# Gitee 响应
HTTP/1.1 200 OK
Cache-Control: no-cache, max-age=0, must-revalidate
Connection: keep-alive
Content-Type: application/x-git-upload-pack-advertisement
Expires: Fri, 01 Jan 1980 00:00:00 GMT
Pragma: no-cache
Server: nginx
X-Frame-Options: DENY
X-Gitee-Server: Brzox/3.2.3
X-Request-Id: 96e0af82-dffe-4352-9fa5-92f652ed39c7
Transfer-Encoding: chunked

001e# service=git-upload-pack
0000
010fca6ce400113082241c1f45daa513fabacc66a20d HEADmulti_ack thin-pack side-band side-band-64k ofs-delta shallow deepen-since deepen-not deepen-relative no-progress include-tag multi_ack_detailed no-done symref=HEAD:refs/heads/testbody object-format=sha1 agent=git/2.29.2
003c351bad7fdb498c9634442f0c3f60396e8b92f4fb refs/heads/dev
004092ad3c48e627782980f82b0a8b05a1a5221d8b74 refs/heads/dev-pro
0040ae747d0a0094af3d27ee86c33e645139728b2a9a refs/heads/develop
0000
```

我们需要关注的信息在 Header 和 Body，这里简单介绍一下，更详细的介绍请参见上面提到的`http-protocol.txt`文档
Header 包含了一些约定：

1. Cache-Control 必须禁止缓存，不然可能看不到最新的提交信息
2. Content-Type 必须是`application/x-$servicename-advertisement`，不然客户端会以哑协议的方式去处理
3. 客户端需要验证返回的状态码，如果是`401`那么就提示输入用户名密码

另外我们能看到返回的 Body 格式跟哑协议所用的`info/refs`内容是不一样的，这里是智能协议所约定的格式，客户端根据这个来识别支持的属性和验证信息，这是一个`pkt-line`格式的数据：
1. 客户端需要验证第一首行的四个字符符合正则`^[0-9a-f]{4}#`，这里的四个字符是代表后面内容的长度
2. 客户端需要验证第一行是`# service=$servicename`
3. 服务端得保证每一行结尾需要包含一个`LF`换行符
4. 服务端需要以`0000`标识结束本次请求响应

在`HEAD`引用后还有一系列的服务端能力的参数，这些参数会告诉客户端服务端具有什么样的能力，比如可以通过`multi_ack`模式进行数据交互等，这里不在赘述。再往后就是具体的每一个引用的信息，每行的开头四个字符均是本行的长度。

在介绍哑协议的时候，我们使用通过`git update-server-info`命令生成的`info/refs`文件，但是很明显我们在智能协议这里无法直接使用，因为它不符合`pkt-line`的格式，所以 Git 提供另外一种方式：通过 `git upload-pack`命令来直接获取最新的`pkt-line`格式的引用信息，来看看它的参数支持：

```

--[no-]strict
Do not try <directory>/.git/ if <directory> is no Git directory.

--timeout=<n>
Interrupt transfer after <n> seconds of inactivity.

--stateless-rpc
Perform only a single read-write cycle with stdin and stdout. This fits with the HTTP POST request processing
model where a program may read the request, write a response, and must exit.

--advertise-refs
Only the initial ref advertisement is output, and the program exits immediately. This fits with the HTTP GET
request model, where no request content is received but a response must be produced.

<directory>
The repository to sync from.
```

`upload-pack`是用来发送对象给客户端的一个远程调用模块，但是它提供了`--stateless-rpc`和`--advertise-refs`参数，能够让我们快速拿到当前的引用状态并退出，我们在服务端的裸仓库目录执行就可以直接拿到最新的引用信息：

```
➜  .git git:(master) git upload-pack --stateless-rpc --advertise-refs .
010aef8021acf4c29eb35e3084b7dc543c173d67ad2a HEADmulti_ack thin-pack side-band side-band-64k ofs-delta shallow deepen-since deepen-not deepen-relative no-progress include-tag multi_ack_detailed no-done symref=HEAD:refs/heads/master agent=git/2.24.3.(Apple.Git-128)
003fef8021acf4c29eb35e3084b7dc543c173d67ad2a refs/heads/master
0000%
```

这里的内容是不是似曾相识，跟上面我们抓包获取到的 Gitee 返回的引用数据格式一样，只是少了首行的`# service=git-upload-pack`，所以我们现在思路非常清晰，可以先来实现第一步引用发现的服务端的处理，通过对参数的解析，我们可以拿到仓库名称以及相应的操作名称，就可以进一步整理出客户端要的响应格式：

```golang
// 支持 upload-pack 和 receive-pack 两种操作引用发现的处理
func handleRefs(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	repoName := vars["repo"]
	repoPath := fmt.Sprintf("%s%s", *repoRoot, repoName)
	service := r.FormValue("service")
	pFirst := fmt.Sprintf("# service=%s\n", service) // 本示例仅处理 protocol v1

	handleRefsHeader(&w, service) // Headers 处理

	cmdRefs := exec.Command("git", service[4:], "--stateless-rpc", "--advertise-refs", repoPath)
	refsBytes, _ := cmdRefs.Output() // 获取 pkt-line 数据
	responseBody := fmt.Sprintf("%04x# service=%s\n0000%s", len(pFirst)+4, service, string(refsBytes)) // 拼接 Body

	_, _ = w.Write([]byte(responseBody))
}

// 按照要求设置 Headers
func handleRefsHeader(w *http.ResponseWriter, service string) {
	cType := fmt.Sprintf("application/x-%s-advertisement", service)
	(*w).Header().Add("Content-Type", cType)
	(*w).Header().Set("Expires", "Fri, 01 Jan 1980 00:00:00 GMT")
	(*w).Header().Set("Pragma", "no-cache")
	(*w).Header().Set("Cache-Control", "no-cache, max-age=0, must-revalidate")
}
```

上面我们也提到了，无论是拉取还是推送，都需要先进行引用发现，实际上`upload-pack`和`receive-pack`所处理的差别仅仅是调用的命令不同而已，这一点我们也在`handleRefs`函数里面做了相应的兼容处理，这里不再赘述。

#### 2. 数据传输

数据传输分为两部分：客户端向服务端传输（Push）、服务端向客户端传输（Fetch）。
两者的区别在于：
1. Fetch 操作在获取引用发现之后，由 **服务端** 计算出客户端想要的数据，并把数据以`pkt-line`的格式`POST`给服务端，由服务端进行`Pack`的计算和打包，将包作为 `POST` 的响应发送给客户端，客户端进行解压和引用更新
2. Push 操作获取到服务端的引用列表后，由 **客户端** 本地计算出客户端所缺失的数据，将这些数据打包，并`POST`给服务端，服务端接收到后进行解压和引用更新

Fetch 操作其实用到了上面我们提到的`upload-pack`，它是一个发送对象给客户端的远程调用模块，为了实现拉取功能，我们只需要在服务端启动 `git upload-pack --stateless-rpc` ，这个命令阻塞的接收一串参数，而这串参数是客户端的第二次请求发送过来的，把它传递给这个命令，Git 就会自动的计算客户端所需要的最小对象集并打包，以流的形式返回这个包数据，我们只需要把这个包作为 POST 请求的响应发给客户端就好了。

那么，在 Fetch 操作中，客户端第二次 POST 请求发过来的数据是什么呢，我们也来抓个包分析一下：

```
POST http://gitee.com/kesin/bigRepo/git-upload-pack HTTP/1.1
Host: gitee.com
User-Agent: git/2.24.3 (Apple Git-128)
Accept-Encoding: deflate, gzip
Proxy-Connection: Keep-Alive
Content-Type: application/x-git-upload-pack-request
Accept: application/x-git-upload-pack-result
Content-Length: 443

00b4want bee4d57e3adaddf355315edf2c046db33aa299e8 multi_ack_detailed no-done side-band-64k thin-pack include-tag ofs-delta deepen-since deepen-not agent=git/2.24.3.(Apple.Git-128)
00000032have 82a8768e7fd48f76772628d5a55475c51ea4fa2f
0032have 4f7a2ea0920751a5501debbbc1debc403b46d7a0
0032have 7c141974a30bd218d4257d4292890a9008d30111
0032have f6bb00364bd5000a45185b9b16c028f485e842db
0032have 47b7bd17fcb7de646cf49a26efb43c7b708498f3
0009done
```
客户端在拿到第一次引用发现服务端返回的数据后，会根据服务端所提供的能力(`capabilities `)列表以及引用（`refs`）列表来计算出第二次需要发送的数据，比如会根据服务端的能力列表来决定客户端和服务端通信所需要的能力参数，这些能力参数服务端必须全部支持。另外，客户端发送的数据必须包含一个`want`指令，我们在 Clone 一个仓库的时候所发送的数据全部都是`want`指令，而不包含`have`指令，因为本地什么都没有；而在进行有数据的更新的 Fetch 操作的时候，就会有`have`指令。客户端会根据返回的引用信息计算出所需要的 `Commit`、`Common Commit` 以及 自己有服务端没有的 `Commit`，并将这些数据一次性的通过第二次请求发给服务端，具体客户端的协商过程可以参见`http-protocol.txt`，这里不再赘述。

服务端在收到这些数据之后，会先确认`want`指令所指定的对象是否都能够在引用中找到，如果没有`want`指令或者指令指定的对象中有不包含在服务端的，则会返回给客户端错误信息，服务端根据这些信息计算出客户端所需要的对象的集合，并把这些对象打包返回给客户端，客户端接收后解压包并更新引用。

Push 操作大同小异，只不过在第二步的时候，客户端会根据服务端的引用信息计算出服务端所需要的对象，直接通过 Post 请求发送给服务端，并同时附带一些指令信息，比如新增、删除、更新哪些引用，以及更新前后的版本，具体格式如下：

```golang
   // https://github.com/git/git/blob/master/Documentation/technical/http-protocol.txt#L474
   C: POST $GIT_URL/git-receive-pack HTTP/1.0
   C: Content-Type: application/x-git-receive-pack-request
   C:
   C: ....0a53e9ddeaddad63ad106860237bbf53411d11a7 441b40d833fdfa93eb2908e52742248faf0ee993 refs/heads/maint\0 report-status
   C: 0000
   C: PACK....
```
这里的包数据格式为`"PACK" <binary data>`，会以`PACK`开头。服务端接收到这些数据后，启动一个远程调用命令`receive-pack`，然后将数据以管道的形式传给这个命令即可。

所以，整个数据传输的过程无非就是客户端与服务端的`upload-pack`和`receive-pack`对规定格式的数据交换而已，根据这个思路，我们可以继续完善我们的 Smart Git HTTP Server，来增加对第二步的处理能力：

```golang
func processPack(w http.ResponseWriter, r *http.Request) {
	vars := mux.Vars(r)
	repoName := vars["repo"]
	// request repo not end with .git is supported with upload-pack
	repoPath := fmt.Sprintf("%s%s", *repoRoot, repoName)
	service := vars["service"]

	handlePackHeader(&w, service)

	// 启动一个进程，通过标准输入输出进行数据交换
	cmdPack := exec.Command("git", service[4:], "--stateless-rpc", repoPath)
	cmdStdin, err := cmdPack.StdinPipe()
	cmdStdout, err := cmdPack.StdoutPipe()
	_ = cmdPack.Start()

	// 客户端和服务端数据交换
	go func() {
		_, _ = io.Copy(cmdStdin, r.Body)
		_ = cmdStdin.Close()
	}()
	_, _ = io.Copy(w, cmdStdout)
	_ = cmdPack.Wait() // wait for std complete
}
```

完整的实现见：[https://gitee.com/kesin/go-git-protocols/tree/master/http-smart](https://gitee.com/kesin/go-git-protocols/tree/master/http-smart)

## Git && SSH 传输协议

Git 协议以及 SSH 协议都是四层的传输协议，而 HTTP 则是七层的传输协议，受限于 HTTP 协议的特点，HTTP 在 Git 相关的操作上存在传输限制、超时等问题，这个问题在大仓库的传输中尤为明显，相比与 HTTP 而言，Git 以及 SSH 协议在传输上更简单而且更稳定。

> 官方文档：[https://github.com/git/git/blob/master/Documentation/technical/pack-protocol.txt](https://github.com/git/git/blob/master/Documentation/technical/pack-protocol.txt)

### Git 协议

Git 协议最大的优势就是速度快，因为它没有 HTTP 的传输协议的条条框框，也没有 SSH 加解密的成本，但受限于协议的缺点，Git 协议常用于开源项目的下载，不作为私有项目的传输协议。

在上面我们研究 HTTP 智能协议的实现的时候，我们知道 Git 客户端跟服务端的交互主要包含两个步骤：
1. 获取服务端的引用
2. 客户端根据服务端的引用数据与服务端进行数据交换

Git 协议也是如此，只不过相比于 HTTP 协议，Git 协议直接在四层与服务端建立连接，通过这个长链接直接完成两个步骤：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0124/204919_ea8266e7.png "在这里输入图片标题")

在使用 Git 协议操作的时候，首先客户端会把相关的信息发给服务端，这个信息的格式同样的采用`pkt-line`的格式：

```
003egit-upload-pack /project.git\0host=myserver.com\0\0version=1\0
```

其中包含了命令、仓库名称、Host 等相关信息，服务端建立连接之后，接收到这串信息，需要对其中的信息进行加工，找到对应的仓库所在的位置也就是目录，当所有的信息都符合要求之后，只需要在服务端启动`upload-pack`命令即可，这里需要注意的是我们不需要添加`--stateless-rpc`参数，直接`git upload-pack {repo_path}`，这个命令启动后会马上返回相关的引用信息并且阻塞等待下一次信息的输入：

```shell
➜  hello git:(master) ✗ git upload-pack .
010234d8ed9a9f73d2cac9f50a8c8c03e4643990a2bf HEADmulti_ack thin-pack side-band side-band-64k ofs-delta shallow deepen-since deepen-not deepen-relative no-progress include-tag multi_ack_detailed symref=HEAD:refs/heads/master agent=git/2.24.3.(Apple.Git-128)
003f34d8ed9a9f73d2cac9f50a8c8c03e4643990a2bf refs/heads/master
0000
```

这个时候我们所做的其实就是数据的转发，命令的标准输出信息我们原封不动的发送给客户端，客户端则会进行跟 HTTP 协议类似的处理产生数据，接着会把数据发给服务端，我们再原封不动的发给`git upload-pack {repo_path}`命令的标准输入，然后服务端处理完成后会把相应的包通过标准输出返回，我们原封不动的发给客户端，就完成了一次 Fetch 操作，而 Push 的 `receive-pack` 操作原理相同，这里不再赘述。

需要注意的是，如果客户端发送的信息不符合要求，或者处理过程中出现了问题，我们返回错误告知客户端，这个错误的格式也是`pkt-line`格式的，以`ERR`开头：

```
// error-line     =  PKT-LINE("ERR" SP explanation-text)
func exitSession(conn net.Conn, err error) {
	errStr := fmt.Sprintf("ERR %s", err.Error())
	pktData := fmt.Sprintf("%04x%s", len(errStr)+4, errStr)
	_, _ = conn.Write([]byte(pktData))
	_ = conn.Close()
}
```
客户端接收到这个信息之后，就会打印信息并关闭连接，整个过程的数据均可以通过转包获取到，感兴趣的同学可以通过抓包来进一步加深了解 Git 协议的传输过程。

了解了 Git 协议的过程之后，我们就可以通过代码来实现一个简单的 Git 协议服务器：

```golang
func handleRequest(conn net.Conn) {
    //处理首次客户端发来的数据，拿到 action 以及 仓库信息
    service, repo, err := parseData(conn) 

    // 仅支持 Push 和 Fetch 操作
    if service != "git-upload-pack" && service != "git-receive-pack" {
		exitSession(conn, errors.New("Not allowed command. \n"))
	}
	repoPath := fmt.Sprintf("%s%s", *repoRoot, repo)

	cmdPack := exec.Command("git", service[4:], repoPath)
	cmdStdin, err := cmdPack.StdinPipe()
	cmdStdout, err := cmdPack.StdoutPipe()
	_ = cmdPack.Start()

    // 客户端服务端数据交换
	go func() { _, _ = io.Copy(cmdStdin, conn) }()
	_, _ = io.Copy(conn, cmdStdout)
	err = cmdPack.Wait()
}
```

完整的实现见：[https://gitee.com/kesin/go-git-protocols/tree/master/git-server](https://gitee.com/kesin/go-git-protocols/tree/master/git-server)

### SSH 协议

SSH 协议也是应用的比较广泛的一种 Git 传输协议，相比于 Git 协议，SSH 协议从数据传输和权限认证上都相对安全，但是受限于加解密的成本，速度会稍慢，但是这个时间成本在安全面前绝对是可以接受的。与 Git 协议比较，不同点是 SSH 协议传输的数据经过加密，相同点是 SSH 协议的传输过程与 Git 协议一致，都是跟服务端的进程做数据交换：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0124/230447_775ca76a.png "在这里输入图片标题")

SSH 的下载地址一般都是 `git@gitee.com:kesin/go-git-protocols.git` 这种形式的，在执行 Clone 或者 Push 的时候，会拆解成：
```
ssh user@example.com "git-upload-pack '/project.git'"
```
所以 SSH 协议在首次传参的时候与 Git 协议的格式不同，其他情况基本一致，比如引用发现、Packfile 机制、错误处理等等，这里都不再做延伸，可以参加官方文档。

> 对 Packfile 的协商生成策略感兴趣的可以参见：[https://github.com/git/git/blob/master/Documentation/technical/pack-protocol.txt#L229](https://github.com/git/git/blob/master/Documentation/technical/pack-protocol.txt#L229)

明白 SSH 协议是怎么回事后，我们想要实现一个 Git SSH 服务器就比较明确了，只需要实现一个 SSH Server 并且在对应的 Session 做对应的数据传输即可，我们来实现一个简单的 Git SSH 服务，代码如下：

```
func main() {
	// init host key and public key authentication
	var hostOption ssh.Option
	hostOption = ssh.HostKeyFile(*hostKeyPath)
	keyHandler := func(ctx ssh.Context, key ssh.PublicKey) bool {
		// replace your public key auth logic here
		pubKeyStr := gossh.MarshalAuthorizedKey(key)
		return true // set false to use password authentication
	}
	keyOption := ssh.PublicKeyAuth(keyHandler)

	// password validate authentication
	pwdHandler := func(ctx ssh.Context, password string) bool {
		// replace your own password auth logic here
		if ctx.User() == "zoker" && password == "zoker" {
			return true
		}
		return false
	}
	pwdOption := ssh.PasswordAuth(pwdHandler)

	// process ssh session pack
	ssh.Handle(func(s ssh.Session) {
		handlePack(s) // 处理请求
	})

	addr := fmt.Sprintf(":%s", *port)
	log.Printf("Starting ssh server on port %s \n", *port)
	log.Fatal(ssh.ListenAndServe(addr, nil, hostOption, pwdOption, keyOption))
}

func handlePack(s ssh.Session) {
	args := s.Command()
	service := args[0]
	repoName := args[1]
    // allowed command
	if service != "git-upload-pack" && service != "git-receive-pack" {
		exitSession(s, errors.New("Not allowed command. \n"))
	}

	repoPath := fmt.Sprintf("%s%s", *repoRoot, repoName)
    // 启动标准输入输出进行数据交换，下面的处理是否似曾相识？没错，Git 协议也是同样的处理方式
	cmdPack := exec.Command("git", service[4:], repoPath)
	cmdStdin, err := cmdPack.StdinPipe()
	cmdStdout, err := cmdPack.StdoutPipe()
	_ = cmdPack.Start()
	go func() { _, _ = io.Copy(cmdStdin, s) }()
	_, _ = io.Copy(s, cmdStdout)
	_ = cmdPack.Wait()
}
```

完整的实现见：[https://gitee.com/kesin/go-git-protocols/tree/master/ssh-server](https://gitee.com/kesin/go-git-protocols/tree/master/ssh-server)

### 总结

Git 传输协议还有很多进阶的用法由于篇幅的限制未介绍到，比如 Packfile 协商机制、Shallow mode、Protocol V2等等，基于这些机制我们可以发挥想象力，实现具有特定功能的 Git 服务来为某一场景的用户服务，更多的请研读官方文档，没有什么比官方文档更全面了。Git 是一个非常好的版本控制系统，相信未来它会有更广泛的应用，也会有更多的功能推出来适应飞速发展的研发效能体系。

### 最后

本文在2020年底就计划要写，记得当时跟公司一个兄弟`五环`在客户那里处理事情，忙完已经月残人稀，等车的途中聊到我要在元旦写篇 Git 传输协议的文章，来纪念这猝不及防的30岁，怎奈元旦到了最后一天才写了一半，紧接着就是铺天盖地的各项事务，把这篇文章忘的一干二净，好在及时在1月底的倒数第二个周末完成，也算是亡羊补牢了。

希望今后量力而行，做不到的不乱说，说出来的要做到，对于计划可能的偏离要有预备方案和追赶措施，人生果然如架构，亦如项目管理，连调优都如此雷同，也许为的就是能够承受更大的「压力」和「意外」吧。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0125/003521_d71b3ff1.png "在这里输入图片标题")

转载请保留出处：

微信公众号「Zoker 随笔」（zokersay）