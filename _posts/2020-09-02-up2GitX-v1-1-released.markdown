---
layout: post
title: up2GitX v1.1.0 发布 - 支持镜像 Github 空间到 Gitee
date: 2020-09-02 09:30:41
excerpt: 本次更新发布应呼声，支持选择上传到 Gitee 组织，支持创建组织，并上传仓库到创建的组织，支持镜像 Github 空间（个人或组织）到 Gitee 的组织或个人空间下。
---

up2GitX 是一个方便快捷的批量 Git 托管工具，将本地仓库批量上传至 Gitee、Github、Gitlab 平台

### v1.1.0 更新说明
- 支持选择上传到 Gitee 组织
- 支持创建组织，并上传仓库到创建的组织
- 支持镜像 Github 空间（个人或组织）到 Gitee 的组织或个人空间下

### 下载
下载对应平台的二进制包，可直接运行
- [up2-macos-v1.1.0.zip](https://gitee.com/kesin/up2GitX/attach_files/432062/download)
- [up2-linux-v1.1.0.zip](https://gitee.com/kesin/up2GitX/attach_files/432063/download)

### 使用说明

```
  Using Source:  ./up2 gitee github:zoker YOUR_TOKEN_HERE(OPTIONAL)
  Support import from Github source, replace {zoker} with your expected Github path
  It better to provide your own access_token to avoid api rate limit, eg on Github: https://github.com/settings/tokens
  Alert: Only Github source and public project supported, other platform like Gitlab, Bitbucket will be added later
```

使用 `up2 gitee github:oschina xxxxxx` 命令进行镜像上传，其中xxxxx为 Github 的 Personal Token，是为了防止 API 请求超限，如果不设置，会有可能超限而被屏蔽。Github Personal Token 申请地址：https://github.com/settings/tokens

#### 示例：

以镜像`oschina`的 Github 仓库为例

##### 1、首先输入命令进行仓库列表的拉取

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0823/095053_46e422f9.png "在这里输入图片标题")

命令会先判断`oschina`的类型，根据类型是个人还是组织进行仓库列表的拉取，一次拉取20个，直到拉取完所有的仓库为止

##### 2、确认是你想要的仓库，输入 Gitee 的用户名密码进行鉴权

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0823/095102_e873848b.png "在这里输入图片标题")

鉴权通过后，会有列表可以选择是上传到个人下，还是上传到组织下，还是上传到一个新的组织，这里我们选择上传到新的组织

##### 3、输入新组织的名称并创建

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0823/095110_7d4373e0.png "在这里输入图片标题")

如果组织名称符合规则，则会提示创建成功，然后会给出将要创建的 Gitee 的仓库列表

##### 4、选择将要创建的 Gitee 仓库的属性

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0823/095117_97b9c9af.png "在这里输入图片标题")

这里我们选择公开

##### 5、创建完成后，会列出创建成功、已经存在或者创建失败的列表

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0823/095152_175299a4.png "在这里输入图片标题")

根据提示选择对应的操作，这里我们选择跳过

##### 6、进入 Clone and Push 的阶段

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0823/095159_78372afd.png "在这里输入图片标题")

到了这里基本就完成了，只需要等待本地 Clone 及 上传到 Gitee 即可，这里会在命令执行目录创建一个`up2GitX-github-oschina`临时目录来存储这些临时 Clone 下来的裸仓库，待同步完成后，删除即可

### 注意事项

如果失败或者中断，只需要重试，对已经存在的仓库选择覆盖操作即可。