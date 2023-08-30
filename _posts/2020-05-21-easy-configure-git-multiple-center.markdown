---
layout: post
title: 通过配置实现极简的 Git 服务的异地多中心方案
date: 2020-05-21 14:19:06
excerpt: 最近遇到一个场景，客户希望自己在多个地区的开发科室能够共用一套Git系统，但是由于跨地区的原因，带宽是这种使用场景的唯一障碍，增量的更新还勉强可以接受，但是如果需要全量Clone一个上G的仓库，那将会非常非常慢。限于有限的开发资源和众多的客户需求，于是思考了下可以通过配置以及少量的开发即可实现简单的异地多中心方案。
---

### 背景

最近遇到一个场景，客户希望自己在多个地区的开发科室能够共用一套Git系统，但是由于跨地区的原因，带宽是这种使用场景的唯一障碍，增量的更新还勉强可以接受，但是如果需要全量Clone一个上G的仓库，那将会非常非常慢。

这种场景其实通过对组件的路由以及同步策略进行二次开发其实可以完美解决，但是限于有限的开发资源和众多的客户需求，只能找寻临时的解决方案，于是思考了下可以通过配置以及少量的开发即可实现简单的异地多中心方案，解决用户由于异地拉取造成的缓慢问题，实现就近读、统一写的功能，本方案适用于仅有一台后端Git服务器的场景。

这里的Git服务后端可以是Gitee、Gitlab、Gogs等有Web管理界面的应用，当然一个纯粹的Git服务端也是可以的，这里只提供一种思路，不做实际应用的探讨。

### 实现概述

整套系统可采用统一的数据库及其他依赖的三方服务，因为流量压力并不在数据库上，而是在频繁的Clone、Pull操作上，所以通过对Git操作的拆解，可以实现写统一到中心节点，而读则可以就近读，通过中心节点的实时文件同步则可以做到从节点的文件一致性（iNotify+rsync）

常见的Git服务系统可以写到仓库的一般分为以下几种操作  

- 通过Web操作进行仓库的写（一般情况下都是Post请求，根据实际情况而定）  
- 通过http进行推送（Post请求）  
- 通过ssh方式进行推送（sshd服务处理post-receive请求）  

所以我们只需要通过nginx配置处理http请求，通过更改ssh服务的逻辑更改即可实现。

![image](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0521/224853_28efd0de.png "架构图")

### 具体实现配置

首先简单了解一下Git的两个操作机制 receive-pack & upload-pack

- receive-pack: 为推送的操作，当你执行 git push 的时候，服务器就会唤起 git-receive-pack来进行传输对象的接收
- upload-pack: 同样的，当你执行 git pull 或者 git fetch 的时候，服务器就会唤起 git-upload-pack 进行对象的传输

**下面分两种操作方式进行实现的介绍： http & ssh** 

首先这里假设我们有两个地区A&B

- A地区服务器的IP是 10.1.10.100 ; nginx 端口是 80 ; Gitee Web 端口是3000
- B地区服务器的IP是 10.1.10.200 ; nginx 端口是 80 ; Gitee Web 端口是3000

#### HTTP 请求的读写拆离

A地区服务器的服务正常配置即可，所有A地区的Git服务域名均解析到10.1.10.100

B地区的服务器只需要将仓库的写请求回源到A地区即可，所有B地区的Git服务域名均解析到10.1.10.200，B地区的nginx服务的配置如下：

```
    # 有get操作会写仓库的需要单独配置回源到IP
    location ~^/(.+)/(.+)/xxx/.+{
        proxy_pass http://10.1.10.100:3000;
        proxy_set_header Accept-Encoding "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-Id $request_id;
    }

    # git http 推送操作，回源到A地区（如果没有Web服务会动到仓库，仅配置此项即可，下面的根据Post回源的配置可以不需要
    location ~^/(.+)/(.+)/git-receive-pack$ {
        proxy_pass http://10.1.10.100:3000;
        proxy_set_header Accept-Encoding "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-Id $request_id;
    }

    # get 请求全部就近到B地区IP，post请求回源到A地区
    location / {
        proxy_set_header Accept-Encoding "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Request-Id $request_id;
        if ($request_method = 'GET') {
            proxy_pass http://10.1.10.200:3000;
            break;
        }
        proxy_pass http://10.1.10.100:3000; 
    }
```
配置完成后即可实现B地区的用户通过http的操作仓库全部回源到A地区的服务器，并且A地区的服务器通过同步机制，将仓库同步回B地区的服务，B地区用户的读操作会完全在B地区的服务器进行处理。

#### SSH 请求的读写拆离

SSH 请求的改造需要根据系统的SSH组件来进行对应的改造，目前常用的是两种：
- 通过OPENSSH进行鉴权和传输链接的建立（通常通过API获取鉴权信息和路由，通过远程命令的方式与路由机器建立链接，Gitlab 老版本均采用此种方式
- 通过SSH组件分发到后端Git服务进行实现（通常通过API获取鉴权信息和路由选择与哪个后端Git服务建立链接

两种方式均需要对对应的路由逻辑进行调整，实现的方式是提供配置项Localip

A地区服务：无需配置Localip，因为默认都是回源到A地区的服务的
B地区服务：需要配置Localip为B地区的IP10.1.10.200

然后通过传过来的参数进行读写的识别

- if receive-pack
返回A地区的IP，让组件与A地区的Git服务建立链接传输数据
- elsif upload-pack
返回本地区的IP，让组件与本地区的Git服务建立链接传输数据，速度是最快的
- end

需要注意的是默认情况下，所有地区的仓库分配的IP都应该是A地区的IP

#### 仓库的同步

同步不做过多描述，可以采用rsync定时同步，这里提供一个3s同步一次的脚本

```
#/bin/bash
while true
do
    # 可同步多个地区
    sshpass -p 123456 rsync -av -e 'ssh -p 222' `readlink -f /app/repositories/` git@10.1.10.200:/app/repositories/
    sshpass -p 123456 rsync -av -e 'ssh -p 222' `readlink -f /app/repositories/` git@10.1.10.300:/app/repositories/
    sshpass -p .......
    # 时间间隔
    sleep 3
done
```
另外一种同步的方案就是相对来说比较实时的同步方案
iNotify+rsync，可以参见：[使用iNotify+rsync实现秒级文件同步](https://zoker.io/blog/inotify-and-rsync-file-sync
)

(END)