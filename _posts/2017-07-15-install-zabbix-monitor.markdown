---
layout: post
title: 企业级监控 Zabbix 之安装与使用
date: 2017-07-15 12:23:59
excerpt: （2015年4月发表于开源中国博客）Zabbix 是一个基于WEB界面的提供分布式系统监视以及网络监视功能的企业级的开源解决方案。目前码云的服务器均使用Zabbix进行集群监控，写这篇博客也是为了记录安装使用过程中的一些总结。
---

Zabbix 是一个基于WEB界面的提供分布式系统监视以及网络监视功能的企业级的开源解决方案。目前码云的服务器均使用Zabbix进行集群监控，Zabbix的使用分为Server和Client。

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2017/0715/220626_7df7c362.png "在这里输入图片标题")

### Zabbix Server

####安装zabbix server

这里Server以Ubuntu系统为例，采取最原始的安装方法

ubuntu的库里面是有zabbix的源的，但是跟不上最新的版本了，所以要zabbix的源添加进去
```
sudo vi /etc/apt/sources.list
```
添加下面两行
```
deb http://ppa.launchpad.net/tbfr/zabbix/ubuntu precise main
deb-src http://ppa.launchpad.net/tbfr/zabbix/ubuntu precise main
```
保存退出

然后需要加上PPA的key，否则apt-get不会信任源
```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys C407E17D5F76A32B
```
安装zabbix server
```
sudo apt-get install zabbix-server-mysql php5-mysql zabbix-frontend-php
```
配置zabbix server，配置文件路径 `/etc/zabbix/zabbix_server.conf`
```
DBName=zabbix
DBUser=zabbix
DBPassword=密码
```
保存退出

####配置mysql

进入package目录，解压初始化sql文件
```
cd /usr/share/zabbix-server-mysql/
sudo gunzip *.gz
```
为zabbix创建一个用户
```
create user 'zabbix'@'localhost' identified by '密码'
```
创建一个名为zabbix的数据库
```
create database zabbix;
```
分配权限
```
grant all privileges on zabbix.* to 'zabbix'@'localhost';
```
更新权限
```
flush privileges;
```
下面进行mysql的初始化，使用刚刚解压出来的sql文件
```
mysql -u zabbix -p zabbix < schema.sql

mysql -u zabbix -p zabbix < images.sql

mysql -u zabbix -p zabbix < data.sql
```
####配置PHP
```
sudo vi /etc/php5/apache2/php.ini
```
增加或者修改下面几行
```
post_max_size = 16M
max_execution_time = 300
max_input_time = 300
date.timezone = UTC
```
然后复制zabbix的配置文件
```
sudo cp /usr/share/doc/zabbix-frontend-php/examples/zabbix.conf.php.example /etc/zabbix/zabbix.conf.php
```
同样修改这个文件的数据库配置
```
DBName=zabbix
DBUser=zabbix
DBPassword=密码
```
####配置apache

复制配置文件
```
sudo cp /usr/share/doc/zabbix-frontend-php/examples/apache.conf /etc/apache2/conf.d/zabbix.conf
sudo a2enmod alias
```
然后重启
```
sudo service apache2 restart
```
修改zabbix的初始化文件
```
sudo vi /etc/default/zabbix-server
```
到文件的最后，修改如下
```
START=yes
```
####启动Zabbix-server
```
sudo service zabbix-server start
```

###Zabbix Agent

####安装agent

ubuntu
```
sudo apt-get install zabbix-agent
```

centos
```
rpm -ivh http://repo.zabbix.com/zabbix/2.0/rhel/6/x86_64/zabbix-release-2.0-1.el6.noarch.rpm

yum install zabbix-agent
```

####配置Zabbix agent
```
sudo vi /etc/zabbix/zabbix_agentd.conf
```
只需要修改Server的IP地址即可
```
Server=127.0.0.1 #这里监控自身，就写127.0.0.1即可
```
重新启动
```
sudo service zabbix-agent restart
```
####Web添加Host

进入zabbix监控，用户名和密码默认是admin:zabbix

如下图，点击Create host

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2017/0715/202050_e4b1c153.png "在这里输入图片标题")

然后填写`1、2`的信息，这里提醒一下，本地就不说了，如果另外一台agent，那么需要把10050端口打开，否则没法get到数据

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2017/0715/202117_fef961a0.png "在这里输入图片标题")

进入`3 Templates`

首先输入linux，然后选择第一个 Template linux，这是一个默认的模版，有基础的linux机器状态的监控，之后点击add 然后再点击save

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2017/0715/202205_500d608f.png "在这里输入图片标题")

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2017/0715/202212_85f373ca.png "在这里输入图片标题")

之后进入监控查看图表即可

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2017/0715/202230_9ae1aa67.png "在这里输入图片标题")

OK，基本的配置就是这些，当然还有nginx，mysql ，redis等等的监控都可以通过脚本获取数据进行绘制，还可以设置trigger自动报警等等，zabbix很强大，以后有用到的功能，深入研究接着分享。