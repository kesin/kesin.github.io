---
layout: page
title: Proxy Balance Http(s) 动态代理工具
permalink: /pages/ProxyBalance/
---

> PBS Server is a http/https proxy balancer, used for select best proxies automaticly.

本工具是为了简单快速的实现一个动态 Http(s) 代理而写，项目上最近有用到 HaProxy 进行相关实现，但是还是感觉太重，而且不支持根据代理的质量（比如时延）进行择优，毕竟代理的网络稳定性并不能得到保障，本着简单的事情只需要简单的实现原则，动手写了一个小工具，分享出来给用得到的人。

### 功能介绍

- 提供 PBS 客户端用来调度请求的代理链接
- 提供 PBC 客户端用来快速配置代理节点
- 支持配置多节点及权重（WRR）
- 支持代理节点分组
- 支持动态检测节点并调整权重或移除废弃节点（ing）
- 支持配置单节点最大并发连接数（ing）
- 支持配置最大并发连接后的阻断或阻塞模式（ing）

### 设计草稿

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2021/0102/171504_0a989fd0.png "ProxyBalance 设计流程及功能说明.png")


### 目录结构

```
├── LICENSE
├── Readme.md
├── go.mod
├── go.sum
├── pbc
│   ├── config.go
│   └── pbc.go // pbc client, used for config a proxy
├── pbs
│   ├── checker.go
│   ├── config.go
│   ├── pbs.go // pbs server, used for manage proxies
│   ├── pbs.json
│   ├── pbs.json.example
│   └── proxy.go
└── test
    └── testServer.go // tools for test servers
```

### 使用说明

#### PBS Server
下载编译构建好的 PBS 客户端，执行`./pbs -h`
```
Usage: Config your pbs.json and start pbs with command ./pbs -c ./config.json &

Args:
	-c	Specify a config json
	-h	Help and quit
	-v	Show current version and quit
```

配置`pbs.json`

```
{
  "Port": 7788,   // port for pbs listening
  "ActiveProxyGroups": "all", // select which groups actived, eg: 'all' or 'asia,africa' are same
  "Proxies": {
    "asia": [
      "127.0.0.1:7791 5", // ip:port weight
      "127.0.0.1:7792 1"
    ],
    "africa": [
      "127.0.0.1:7793 1"
    ]
  }
}
```

使用 `./pbs -c ./pbs.json` 启动即可

#### PBC Client

PBC 配置在你的代理节点，执行`./pbc -h`

```
Usage: you can start pbc at port 7799 with command ./pbc -p 7799 &

Args:
	-p	Specify PBC client port to start
	-h	Help and quit
	-v	Show current version and quit
```
使用`./pbc -p 7791` 即可将服务启动在 7791 端口

### 构建说明

项目很简单，你只需要下载代码，然后分别执行
```
go build pbs.go
go build pbc.go
go build test/testServer.go
```
即可得到对应的二进制可执行文件

### 贡献代码

1. Fork 仓库
2. 创建本地分支 (`git checkout -b my-new-feature`)
3. 提交更改 (`git commit -am 'Add some feature'`)
4. 推送到分支 (`git push origin my-new-feature`)
5. 创建一个 Pull Request

### 贡献者

[@Zoker](https://zoker.io)