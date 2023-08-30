---
layout: post
title: 一起来玩 Jenkins:（5）基于 Gitee + Jenkins 的开源项目自动化协作实战
date: 2020-12-06 14:17:24
excerpt: 在开源理念日渐活跃的今天，越来越多的人开始投身于开源，贡献了越来越多的开源项目。而随着时间的推移，更多的人开始为开源项目添砖加瓦，为某一领域的开源项目贡献出自己的力量。贡献者的增多又给开源作者带来不少审核的压力，实际上投身开源的这些开源作者基本都是业余时间来做，并没有太多的时间投入在开源项目上。本文就为了解决这类场景，介绍下如何在 Gitee 上通过 Jenkins 来为自己的开源项目开启自动化协作，旨在将一些开源相关的工作自动化，留出更多的时间投身开源事业。
---

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1207/000447_9efa61c0.png "在这里输入图片标题")

### Gitee & Jenkins 简介

**[码云 Gitee](https://gitee.com)** 是开源中国（OSChina.net）于 2013 年推出的基于 Git 的代码托管和协作开发平台，提供代码托管服务，与开源中国社区资讯、博客、社区等版块相互补充和促进，希望以此更好地为开发者服务、构建更加完善的开源生态。经过超过七年的砥砺发展，已成为国内最大的代码托管平台。

**[Jenkins](https://jenkins.io) **是一款开源 CI&CD 软件，用于自动化各种任务，包括构建、测试和部署软件，支持各种运行方式，可通过系统包、Docker 或者通过一个独立的 Java 程序运行。

### 开源项目协作流程

#### 协作方式

为开源项目贡献代码一般都是使用 PullRequest（MergeRequest）方式，这种方式一般要求用户先将开源仓库 Fork 到自己名下，然后基于自己名下的仓库进行修改提交，最终以 PullRequest 的形式提交到开源项目主仓库，意思就是我请求合并我的这些代码到你的主仓库，来为这个开源项目修复/贡献一些代码，具体的使用可以参见 Gitee 官方文档：https://gitee.com/help/articles/4128

除了常规的 PullRequest 方式，Gitee 还推出了轻量级 PullRequest（简称轻量级PR），这种方式可以摒除繁琐的 Fork 流程，快速的向开源项目提交一个 PullRequest。比如我在阅读 Readme 的时候发现有表述或者拼写错误，这时就可以直接修改 Readme 文件，提交的时候就会自动生成一个轻量级PR，这种操作类似于知识共享平台的纠错功能。详细介绍可以参见：https://gitee.com/help/articles/4291

相比于常规的 PullRequest，Gitee 轻量级PR更加便捷。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/194733_23efc4bf.png "在这里输入图片标题")

#### 痛点

开源项目作者在接收到用户提交第一个的 PullRequest 的时候，心情想必是非常激动的，毕竟这是对自己项目的一种认可。而且根据我个人的观察，一般情况下开源项目的 Star 数量超过100的时候，才有可能收获自己的第一个 PullRequest。

在 PullRequest 数量少的时候，人工完全能够处理的过来，那么，如果是一个非常火爆的开源项目呢？答案是会有非常非常多的热心用户帮忙提交和贡献代码，而且这些 PullRequest 很有可能造成挤压的情况。随着项目的发展，合规性也要逐渐提上日程，对代码提交的规范，甚至 Commit Message 的提交也有可能需要规范化。

目前开源协作中主要的痛点有如下几个：

1. 提交信息不规范，比如中英文混用，重复的提交信息等
2. 提交的代码需要人工检测是否能够通过构建
3. 对于特定的文件需要进行规范的格式化检查
4. 代码质量检查需要反复沟通跟进
5. 贡献者没有详细阅读贡献者协议或者规范

#### 如何解决

目前可以通过 Gitee + Jenkins 配置 PullRequest Flow，实现智能自动化预处理一些场景，减少人力的维护。Gitee 提供了 [Gitee Jenkins Plugin](https://gitee.com/oschina/Gitee-Jenkins-Plugin) 插件，可以与 Jenkins 联动，实现对每一个 PullRequest 进行一些自动化的检查，实现流程图：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/201848_42d0b6a7.jpeg "在这里输入图片标题")

下面以一个小工具 GiteeAutoUpdate（地址：https://gitee.com/kesin/GiteeAutoUpdate ，以下简称 GAU）为例，展开介绍下具体实现的方式   

>  Note: 以下的示例代码以及配置均为 Demo 为目的，并未做任何异常及错误处理，如果需要投产使用，请做好异常及错误处理，避免出现预期外的情况 :)

#### 实现效果图：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/224106_09fe84cb.png "在这里输入图片标题")

### 自动化配置实战

[GAU](https://gitee.com/kesin/GiteeAutoUpdate) 是一个推送到 Gitee 自动更新到其它平台的 Webhook 服务端，通过简单的配置，可以实现推送到 Gitee 的提交自动推送到其他平台的目的。为了能够自动化的对用户提交的 PullRequest 进行一些前置检查，假设我们需要做以下的一些事情：
1. 配置 PullRequest 的与 Jenkins 联动，当有 PullRequest 提交或者更新的时候，触发 Jenkins 进行前置检查
2. 提交 PullRequest 之后，自动提示用户阅读贡献者协议，然后评论`Rebuild`即可进行下一步的检查（你也可以说评论了`Rebuild`即视为同意贡献者协议）
3. 检查 Commit Message 是否有重复的情况（我们要求每一个 Commit Message 都有具体的意义）
4. 验证构建是否成功
5. 检查配置文件格式是否正确（对于一些关键的信息，需要做针对性的检查）
6. 集成 Sonar 质量检测工具，对代码进行检测

#### Jenkins 的配置
Gitee 提供了 [Gitee Jenkins Plugin](https://gitee.com/oschina/Gitee-Jenkins-Plugin) 插件，可以在 Jenkins 的插件市场搜索进行安装，安装完成后，我们新建一个 Free Style 的工程，命名为 GAU

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/203517_630ebcd6.png "在这里输入图片标题")

在 Source Code Management 配置 Git 源，作如下配置：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/210745_877af174.png "在这里输入图片标题")

**在 Build Triggers 内，选择作如下配置：**

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/210927_6fbc77a9.png "在这里输入图片标题")

**并将 Gitee WebHook URL 添加到对应仓库的 WebHook 中**

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/211257_a6f7761a.png "在这里输入图片标题")

其中 Webhook Token 即在 Jenkins 的 Secret Token for Gitee WebHook 生成的，这里需要注意的是，如果你的 Jenkins 暴露在公网，你可能将 Jenkins 设置了需要权限才可以访问，那么 Gitee 将无法请求你的这个 WebHook，你需要使用插件`Role-Based Strategy` 来配置匿名用户对 Build 的权限。不过不用担心，你可以结合刚刚生成的 Token 来保证安全触发。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/211637_e74fb86e.png "在这里输入图片标题")

接着配置一个构建阶段 Build，我们选择 `Execute shell`，并先填写一个简单的`Hello World`

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/211946_e44311ea.png "在这里输入图片标题")

最后，在 Post-build Actions 配置一下失败后的回传信息，这里我们只需要配置失败后的动作即可，因为其他的情况我们将会通过脚本自行回传。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/212106_40925c62.png "在这里输入图片标题")

提交一个 PullRequest，将会自动触发构建并提示`Triggered ​by ​ZokerBot ​Gitee ​Pull ​Request ​#1: ​kesin/GiteeAutoUpdate ​=> ​master `，并且任务执行也会打出刚刚我们写的那句`Hello World`

> 具体的配置见插件说明：https://gitee.com/oschina/Gitee-Jenkins-Plugin#%E6%8F%92%E4%BB%B6%E9%85%8D%E7%BD%AE

#### 配置 PullRequest 提交后的自动提示信息

有时候，我们需要在用户提交 PullRequest 后，自动添加一句提示或者要求提交者进行一些其他的操作，这里我们假设需要用户阅读贡献者须知:
```
### 贡献代码

1. 提交信息请不要重复，保证每一个提交信息均有意义
2. 提交的代码请添加注释
3. 代码合并后，所有权将遵循项目本身的 LICENSE

### 请求添加同步

1. 请确保已经在对应平台将授权用户加到对应项目
2. 请确保已经添加 Webhook 到你的仓库
3. 请确保`syncWhitelist`内容符合格式
```
为了确保评论信息整洁，我们将内容添加到 Issue 中：https://gitee.com/kesin/GiteeAutoUpdate/issues/I28BNK

那么，我们需要的欢迎信息应该是这个样子的：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/213603_f0c05b2e.png "在这里输入图片标题")

为了快速实现，我们下面使用 Go 写一个脚本`giteeCheck`来实现这个功能：

```
func welcomeMsg(PRIID string) {
	params := make(map[string]interface{})
	params["access_token"] = TOKEN

	if noteNotExist(PRIID) {
		// create first welcome note
		params["body"] = fmt.Sprintf(`欢迎提交 PR，您需要进行如下两步：
						 1. [点击此处阅读贡献者协议及注意事项](https://gitee.com/%s/%s/issues/%s)
						 2. 在本 PR 评论「Rebuild」继续执行构建集成测试状态，表示您阅读并同意了「贡献者协议」
						 `, OWNER, REPO, ISSUE)
		commentUrl := fmt.Sprintf("https://gitee.com/api/v5/repos/%s/%s/pulls/%s/comments", OWNER, REPO, PRIID)
		postForm(commentUrl, params)
		fmt.Printf("0")
		return
	}
	fmt.Printf("1")
}
```
只需要传入当前的 PRIID 即可，为了识别是否是第一次提交，我们判定是否有过评论即可，所以在 Jenkins 的构建过程我们可以添加这么一句命令

```
# add welcome msg when PR created
status=`/home/zoker/app/jenkins/scripts/giteeCheck welcome ${giteePullRequestIid}`
```
其中，${giteePullRequestIid} 为 Gitee Jenkins Plugin 所传入的环境变量，可以在 SHELL 中直接调用，将返回值保存到`status`是为了判定是首次欢迎信息，还是之后的构建。

#### 检查 Commit Message 提交信息是否重复

提交信息是否重复可以根据源分支与目标分支的差异提交来判定，Git 提供了一种方式，可以快速的拿到两个版本之间的差异的提交信息

```
git log HEAD ^remotes/origin/master --pretty=format:'%B'
```

通过去重前后的数量对比，即可得出是否有重复的提交信息，这个过程的命令如下：

```
# check commit msg
  commitCountA=`git log HEAD ^remotes/origin/master --pretty=format:'%B'| sort -r |grep -v '^$'|wc -l`
  commitCountB=`git log HEAD ^remotes/origin/master --pretty=format:'%B'| sort -r |grep -v '^$'|uniq|wc -l`
  if [ $commitCountA == $commitCountB ]; then
      commitMsg="valid"
  else
      commitMsg="invalid"
  fi
```
执行完后，我们将结果保存在`commitMsg`变量，用以后续的信息回传。

#### 验证构建是否成功

这一步检测就比较简单了，只需要执行对应的构建命令，然后判定构建物是否存在即可
```
  # build and check validation
  go build main.go
  if [ -x gau ]; then
      buildStatus="valid"
  else
      buildStatus="invalid"
  fi
```
同样，我们将结果保存在`BuildStatus`，用以后续的信息回传。

#### 检查配置文件`syncWhitelist`格式是否正确

这里检查`syncWhitelist`是为了避免程序启动的时候无法读取到正确的仓库白名单信息而启动失败，类似的行为也可以是团队自研的检查工具或者商业检查工具，用户检查代码格式、注释、是否存在意外上传的 Token 等，这里以检查文件的 Json 格式行为替代。同样的我们写了一个脚本用来解码，保证解码能够正常进行：

```
func checkFile(file string) {
	projects := make(map[string][]string)
	jsonFile, err := os.Open(file)
	if err != nil {
		fmt.Printf("invalid")
		return
	}
	defer jsonFile.Close()

	jsonParser := json.NewDecoder(jsonFile)
	if err := jsonParser.Decode(&projects); err != nil {
		fmt.Printf("invalid")
		return
	}
	fmt.Printf("valid")
}
```
配置到构建命令中：

```
# check syncWhitefile validation
  syncFormat=`/home/zoker/app/jenkins/scripts/giteeCheck checkfile config/syncWhitelist`
```
#### 集成 Sonar 代码质量检测

Sonar (SonarQube)是一个开源平台，用于管理源代码的质量。Sonar 不只是一个质量数据报告工具，更是代码质量管理平台。它的安装和使用都非常简单，只要支持 Java 11 环境即可，这里不在赘述 Sonar 的安装和使用，具体可以参见官方文档：https://docs.sonarqube.org/latest/

我们所需要知道的就是，Sonar 由 SonarWeb 以及 SonarRunner 两部分组成，SonarWeb 用以收集数据及展示，SonarRunner 用于分析仓库代码生成质量数据，只要在对应的仓库下执行一条命令，就可以得到分析报告，我们可以根据这个报告重的数据，来做一些门禁，比如多于5个警告或者有1个严重则直接拒绝掉 PullRequest，让提交者完善后再行提交，可以在一定程度上解放人力。

为了快速实现，我们接下来仅仅关注`warningCount`，只要`warningCount`大于零，就提示用户可能有风险，在执行完分支命令后，我们可以从结果中拿到一个 API 地址，这个地址包含了本次分析的一些基本数据：

```
INFO: Analysis report generated in 50ms, dir size=134 KB
INFO: Analysis report compressed in 15ms, zip size=31 KB
INFO: Analysis report uploaded in 15ms
INFO: ANALYSIS SUCCESSFUL, you can browse http://127.0.0.1:8089/dashboard?id=taskover
INFO: Note that you will be able to access the updated dashboard once the server has processed the submitted analysis report
INFO: More about the report processing at http://127.0.0.1:8089/api/ce/task?id=AXYoHjxxxxxxxxDlfF
INFO: Analysis total time: 2.788 s
INFO: ------------------------------------------------------------------------
INFO: EXECUTION SUCCESS
```
我们要的就是拿到`http://127.0.0.1:8089/api/ce/task?id=AXYoHjxxxxxxxxDlfF`这个地址，然后传给我们的脚本进行进一步的处理：
```
  # Sonar check
  /home/zoker/app/sonar/scanner/bin/sonar-scanner -Dsonar.projectKey=GAU -Dsonar.sources=. -Dsonar.host.url=http://127.0.0.1:8089 -Dsonar.login=410e439472a0626bde66aa7564fc3faa9430f7c6 > sonarRes
  sonarResUrl=`cat sonarRes | grep api/ce | cut -d' ' -f8`
  sleep 3 // 防止 SonarWeb 正在处理中就请求导致的 404
```
至此，几个检测项目均配置完成，并且我们所需要的`status` `syncFormat` `commitMsg` `buildStatus` `sonarResUrl`等均已保存在变量中，接下来，我们就要把这些数据回传到 PullRequest 的评论。

#### 构建过结果回传

结果回传无非就是把刚刚得到的一系列的结果，处理后以固定的格式通过 PullRequest 的评论 API 返回给对应的 PullRequest，我们通过脚本对这些数据进行处理，并且根据具体情况返回具体信息：
```
func resultBack(args []string) {
	var a, b, c, d = "success", "success", "success", "success"
	status := make(map[string]string)
	status["valid"] = ":white_check_mark:"
	status["invalid"] = ":x:"

	// sonar warningCount
	sonarRes, _ := http.Get(args[3])
	sonarResBody, _ := ioutil.ReadAll(sonarRes.Body)

	var sonarJson map[string]map[string]interface{}
	var sonarStatus string
	json.Unmarshal(sonarResBody, &sonarJson)
	if sonarJson["task"]["warningCount"].(float64) == 0 {
		sonarStatus = ":white_check_mark:"
		d = "No Warings"
	} else {
		sonarStatus = ":x:"
		d = fmt.Sprintf("Warings: %f", sonarJson["task"]["warningCount"].(float64))
	}

	// others
	if args[4] == "invalid" {
		a = "请本地构建成功后再行提交"
	}

	if args[5] == "invalid" {
		b = "有重复的提交"
	}

	if args[6] == "invalid" {
		c = "syncWhitelist 格式有误，请确认"
	}

	var botMsg string
	if a == "success" && b == "success" && c == "success" {
		botMsg = "所有检查通过，请 @Ultrim 处理"
	} else {
		botMsg = "检测到有 **未通过项** ，请检查调整后重新提交，将会自动重新触发检查。"
	}

	params := make(map[string]interface{})
	params["access_token"] = TOKEN
	params["body"] = fmt.Sprintf(`自动化检测已完成，请根据结果进行调整：

| 步骤   |  结果  | 备注 |
|-------------|---|-----------------|
|      构建 |  %s  |     %s         |
|      提交信息检查    |  %s  |       %s       |
|      syncWhitelist 格式检查       |  %s |   %s      |
|      Sonar 质量检测       | %s  |         %s       |

%s
`, status[args[4]], a, status[args[5]], b, status[args[6]], c, sonarStatus, d, botMsg)

	commentUrl := fmt.Sprintf("https://gitee.com/api/v5/repos/%s/%s/pulls/%s/comments", OWNER, REPO, args[2])
	postForm(commentUrl, params)

}
```

最终，我们的构建脚本看起来就是这样的：

```
# add welcome msg when PR created
status=`/home/zoker/app/jenkins/scripts/giteeCheck welcome ${giteePullRequestIid}`

if [ $status == "1" ]; then
  # check syncWhitefile validation
  syncFormat=`/home/zoker/app/jenkins/scripts/giteeCheck checkfile config/syncWhitelist`

  # check commit msg
  commitCountA=`git log HEAD ^remotes/origin/master --pretty=format:'%B'| sort -r |grep -v '^$'|wc -l`
  commitCountB=`git log HEAD ^remotes/origin/master --pretty=format:'%B'| sort -r |grep -v '^$'|uniq|wc -l`
  if [ $commitCountA == $commitCountB ]; then
      commitMsg="valid"
  else
      commitMsg="invalid"
  fi

  # build and check format
  go build

  if [ -x gau ]; then
      buildStatus="valid"
  else
      buildStatus="invalid"
  fi

  # Sonar check
  /home/zoker/app/sonar/scanner/bin/sonar-scanner -Dsonar.projectKey=GAU -Dsonar.sources=. -Dsonar.host.url=http://127.0.0.1:8089 -Dsonar.login=410exxxxxxxxxxxxxxxxxxx3faa9430f7c6 > sonarRes
  sonarResUrl=`cat sonarRes | grep api/ce | cut -d' ' -f8`
  sleep 3

  # callback result
  /home/zoker/app/jenkins/scripts/giteeCheck result ${giteePullRequestIid} ${sonarResUrl} $buildStatus $commitMsg $syncFormat
fi
```

#### 验证效果

我们来贡献代码为 GAU 修复一个强制推送的问题，我们新建一个`fix-force-push`分支，在这个分支上修改代码：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/222634_9e1fa703.png "在这里输入图片标题")

完善提交信息`update utils.go fixed force push bug`并提交到分支，接着我们创建一个 PullRequest 

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/223357_d71ebde1.png "在这里输入图片标题")

进到 Jenkins 我们可以看到已经触发了构建：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/223503_63c9e9c7.png "在这里输入图片标题")

而且如我们所期望的，有了欢迎信息

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/223629_4122baf1.png "在这里输入图片标题")

我们评论`Rebuild`接受贡献者协议并开始检查，等待片刻即可看到结果：

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/223828_915c1d72.png "在这里输入图片标题")

可以看到，构建失败了，原因是因为 Jenkins 构建机器没有配置代理，下载依赖包超时了，处理完后，我们重新执行 `Rebuild`

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/1206/224106_09fe84cb.png "在这里输入图片标题")

可以看到，由于代码量比较少，所以 Sonar 也没什么质量问题产生。这里需要说明的是，一般情况下更新代码会自动触发构建，不需要我们手动评论`Rebuild`。

以上就是整个自动化检查的实现过程，示例中 giteeCheck 的代码见：https://gitee.com/kesin/gitee-check

### 总结

在配置了开源项目的自动化协作之后，我们就可以省下很多沟通的时间，同样的，提交者也会快速的得到反馈，避免由于开源作者由于时间问题而响应不及时导致的贡献者的积极性下降。

当然本文中提供的例子还有很多改进的地方：

1. 每一个步骤都可以给 PullRequest 打上对应的标签，如`WaitForCheck`、`PreTestFailed`等
2. 构建失败、检查失败、Sonar 分析结果等也可以添加到备注中
3. 可以根据用户提交的差异文件列表来决定执行哪些步骤
4. 根据 PullRequest 的作者及评论者进行鉴权，只有作者有权限触发 Rebuild
5. ...

由于时间问题，不再过多讨论，大家可以为自己的开源项目构建一套合适的自动化、智能化的协作流程，甚至可以通过这种方式来对 Issue 进行智能化的自动回复，可以发挥想象力，一切皆有可能。

（END）