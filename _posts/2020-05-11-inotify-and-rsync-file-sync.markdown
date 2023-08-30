---
layout: post
title: 使用 iNotify + Rsync 实现简单的秒级文件同步
date: 2020-05-11 14:13:11
excerpt: 最近遇到一个场景，需要实现一个类似于附件的实时同步，文件大多数是二进制的，考虑过使用 `DRBD`，但是感觉用了牛刀，于是想到还有 `iNotify+Rsync` 这么一个方案，简单配置即可使用，稳定性及速度都能够达到要求。
---

### 前言

最近遇到一个场景，需要实现一个类似于附件的实时同步，文件大多数是二进制的，考虑过使用 `DRBD`，但是感觉用了牛刀，于是想到还有 `iNotify+Rsync` 这么一个方案，简单配置即可使用，稳定性及速度都能够达到要求。
### 目的

本片章仅作简单的实践指南，通过阅读此文章，您可以快速配置一个`Linux`下的秒级文件目录同步。

### 工具介绍

#### iNotify
iNotify 是在内核 `2.6.13` 版本中引入的一个新功能，它为用户态监视文件系统的变化提供了强大的支持，允许监控程序打开一个独立文件描述符，并针对事件集监控一个或者多个文件，例如打开、关闭、移动/重命名、删除、创建或者改变属性。

##### 一个简单的 Demo 说明

```
/usr/local/bin/inotifywait -mrq --format '%Xe %w%f' -e modify,create,delete,attrib,move /data/test/
```
此命令用来监控`/data/test`目录下的所有文件的变更事件，`-mrq`是让命令始终保持监听状态，并且递归的监控目录树的变化，然后只打印事件信息出来。`'%Xe %w%f'` 是格式化输出，`-e`参数非常重要，使我们能够指定监听哪些事件，默认是监听所有事件，具体可以参阅`-h`。

这个时候我们在`/data/test`目录下创建一个文件`touch testfile`

```
[root@CentOS74 ~]# /usr/local/bin/inotifywait -mrq --format '%Xe %w%f' -e modify,create,delete,attrib,move /data/test/
CREATE /data/test/testfile
ATTRIB /data/test/testfile
```
与此同时命令已经有输出了，触发了创建以及文件属性变更的事件，这种实时的事件对于我们后续进行同步是非常有帮助的，再也不用依赖于`crontab`或者`while true`了

#### Rsync
Rsync 是 Linux 系统下的数据镜像备份工具。使用快速增量备份工具 Rsync 可以远程同步，支持本地复制，或者与其他SSH、Rsync主机同步。

##### Server 端的配置

这个 Server 端其实就是你要同步的机器，其实可以使用`SSH`进行同步，但是为了安全起见，还是配置一下模块吧。

`vi /etc/rsyncd.conf` 创建配置文件

```shell
# 传输文件使用的用户和用户组
uid = zoker
gid = zoker
# 允许 Chroot，提升安全性
use chroot = yes
read only = no
write only = no
hosts allow = 10.211.55.11
hosts deny = *
max connections = 5
pid file = /var/run/rsyncd.pid
transfer logging = yes
log format = %t %a %m %f %b
log file = /var/log/rsync.log

[data]
path = /home/zoker/tmp/data # 同步目录
list = no # 是否允许列出模块内容
ignore errors
comment = data
auth users = zoker # 模块验证的用户名称
secrets file = /etc/rsyncd.secrets # 模块验证密码文件
```
`vi /etc/rsyncd.secrets` 创建认证文件

```
zoker:111111
```
`chmod 600 /etc/rsyncd.secrets` 更改权限

`rsync --daemon` 启动daemon

##### 客户端的配置

`vi /etc/rsync.passwd` 创建认证文件（内容：111111）

`chmod 600 /etc/rsync.passwd` 更改权限

执行命令进行文件的同步

```
[root@CentOS74 etc]# rsync -avz /data/test/ zoker@10.211.55.10::data --password-file=/etc/rsync.passwd
sending incremental file list
testfile
sent 172 bytes  received 62 bytes  468.00 bytes/sec
total size is 1  speedup is 0.02
```
成功

### 两者结合

提供一个简单的脚本进行各种时间的同步判定

```
#!/bin/bash
src=/data/test                                 # 需要同步的源路径
des=data                                       # 目标服务器上 rsync --daemon 配置发布的名称
rsync_passwd_file=/etc/rsync.passwd            # rsync验证的密码文件
ip1=10.211.55.10                               # 目标服务器1
user=zoker                                     # rsync --daemon定义的验证用户名
cd ${src}                            
/usr/local/bin/inotifywait -mrq --format  '%Xe %w%f' -e modify,create,delete,attrib,move ./ | while read file
do
        INO_EVENT=$(echo $file | awk '{print $1}')
        INO_FILE=$(echo $file | awk '{print $2}')
        echo "-------------$(date)-------------"
        echo $file
        if [[ $INO_EVENT =~ 'CREATE' ]] || [[ $INO_EVENT =~ 'MODIFY' ]] || [[ $INO_EVENT =~ 'CLOSE_WRITE' ]] || [[ $INO_EVENT =~ 'MOVED_TO' ]]
        then
                echo 'CREATE or MODIFY or CLOSE_WRITE or MOVED_TO'
                rsync -avzcR --password-file=${rsync_passwd_file} $(dirname ${INO_FILE}) ${user}@${ip1}::${des}
        fi
        if [[ $INO_EVENT =~ 'DELETE' ]] || [[ $INO_EVENT =~ 'MOVED_FROM' ]]
        then
                echo 'DELETE or MOVED_FROM'
                rsync -avzR --delete --password-file=${rsync_passwd_file} $(dirname ${INO_FILE}) ${user}@${ip1}::${des}
        fi

        if [[ $INO_EVENT =~ 'ATTRIB' ]]
        then
                echo 'ATTRIB'
                if [ ! -d "$INO_FILE" ]
                then
                        rsync -avzcR --password-file=${rsync_passwd_file} $(dirname ${INO_FILE}) ${user}@${ip1}::${des}
                fi
        fi
done
```
此脚本实现了对`/data/test`目录的`modify,create,delete,attrib,move`事件的监控，并根据对应的不同事件进行不同的`rsync`同步

Let us have a try!

在`/data/test`目录新建一个文件
```
[zoker@CentOS74 test]$ touch testfile
[zoker@CentOS74 test]$
```
脚本马上监听到事件，并且根据`CREATE`事件进行`Rsync`文件的同步
```
[zoker@CentOS74 data]$ bash syncInTime.sh
---------------Sat May  9 03:49:17 CST 2020-------------------
CREATE ./testfile
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
sending incremental file list
./
testfile

sent 204 bytes  received 31 bytes  470.00 bytes/sec
total size is 9  speedup is 0.04
```
文件已经同步到 Server 机器
```
zoker@backend1:~/tmp/data$ ls
dir1  readme  testfile
```
#### 思考
- 创建文件的时候会产生`CREATE`及`ATTRIB`事件，有点重复，如何合并？
- 脚本为阻塞模式，没有并发特性，考虑利用`goroutine`特性写一个`goSync`

#### 感谢
- iNotify 下载：https://github.com/inotify-tools/inotify-tools
- Rsync 官网：https://rsync.samba.org/
- 脚本出处：https://www.cnblogs.com/ginvip/p/6430986.html
- iNotify 详细介绍：https://www.cnblogs.com/clsn/p/8022625.html
- Rsync 配置介绍：https://www.jianshu.com/p/bd3ae9d8069c

（END）