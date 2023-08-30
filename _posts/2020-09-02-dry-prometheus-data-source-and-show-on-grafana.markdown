---
layout: post
title: 教你自定义 Promethus 数据源并在 Grafana 展示
date: 2020-09-02 16:44:17
excerpt: 从 MRTG 到 ZABBIX 再到 Prometheus & Grafana，技术的变革迭代之快让我无法想象，会想起用 Zabbix 的时候觉得 Zabbix 这东西真的厉害，还专门注册了个域名 Mabbix.com 想要为 Zabbix 做一个手机客户端，不过很快 Prometheus & Grafana 体系就出来了，这里简单介绍下如何自定义 Metrics
---

#### 什么是 Prometheus?
Prometheus 是由 SoundCloud 开发的开源监控报警系统和时序列数据库(TSDB)，Prometheus 使用 Go 语言开发，是 Google BorgMon 监控系统的开源版本。

#### 什么是 Grafana?
Grafana 是一个跨平台的开源的度量分析和可视化工具，可以通过将采集的数据查询然后可视化的展示，并及时通知。

#### Prometheus & Grafana 能做什么

简单来说，Prometheus 提供数据指标，Grafana 负责对这些数据指标进行特定形式的展示，就像我们去海鲜市场，左边是卖海鲜的，右边是处理烹饪海鲜的，完美 CP 无疑。

这里不再去赘述两个应用的安装，都是 Go 开发的，所以跑起来非常容易，而且基础的配置可以让你在 Grafana 上选择了 Prometheus 为数据源之后快速的能够看到对应的数据，社区里有很多针对不同服务的`exporter`，这里我们仅围绕自定义指标进行说明。

首先我们需要先了解一下 Prometheus 客户端提供的四种核心的度量类型

### 标准度量类型

#### 1、Counter - 累加的指标
Counter 是一个计数器，表示一种累加型指标，该指标只能单调递增或在重新启动时重置为零，比如可以计算累计的请求书、Jenkins 完成任务的成功或者失败的数量。

#### 2、 Gauge - 可增可减的指标
Gauge 是最简单的度量类型，只有一个简单的返回值，可增可减，也可以set为指定的值，Gauge 通常用于反映当前状态，比如当前内存使用情况。

#### 3、Histogram - 自带buckets区间用于统计分布的直方图
Histogram 主要用于在设定的分布范围内(Buckets)记录大小或者次数。

例如请求响应时间：0-100ms、100-200ms、200-300ms、>300ms 的分布情况，Histogram会自动创建3个指标，分别为：

- 事件发送的总次数<basename>_count：比如当前一共发生了2次请求
- 所有事件产生值的大小的总和<basename>_sum：比如发生的2次请求总的响应时间为150ms
- 事件产生的值分布在bucket中的次数<basename>_bucket{le="上限"}：比如响应时间0-100ms的请求1次，100-200ms的请求1次，其他的0次

#### 4、Summary - 数据分布统计图
Summary和Histogram类似，都可以统计事件发生的次数或者大小，以及其分布情况。

Summary和Histogram都提供了对于事件的计数_count以及值的汇总_sum，因此使用_count,和_sum时间序列可以计算出相同的内容。

同时Summary和Histogram都可以计算和统计样本的分布情况，比如中位数，n分位数等等。不同在于Histogram可以通过histogram_quantile函数在服务器端计算分位数。 而Sumamry的分位数则是直接在客户端进行定义。因此对于分位数的计算。 Summary在通过PromQL进行查询时有更好的性能表现，而Histogram则会消耗更多的资源。相对的对于客户端而言Histogram消耗的资源更少。

### 作业和实例
在 Prometheus 中，一个可以拉取数据的端点`IP:Port`叫做一个实例（instance），而具有多个相同类型实例的集合称作一个作业（job）

### 自定义指标的两种方式
1. 通过 `textfiles` 收集器进行收集上报
2. 通过自定义`exporter`提供一个实例进行上报

### 一、使用 `textfiles` 收集器收集内存数据并展示

通过如下命令可以获得内存的可用量

```
top -bn 1 -i -c| head -n 5| tail -n 1| awk '{print $9}'
```

在`node_exporter`目录新建`textfiles`目录
```
mkdir textfiles
```

标准的`textfiles`

```
# HELP get_available_mem get available mem test case
# TYPE get_available_mem gauge
get_available_me 624
```

我们采用了`gauge`可增可减的指标，因为内存是上下浮动变化的，`node_expoter`会根据设置的时间来定时的读取这个文件的内容，所以我们只需要通过结合`crontab`来定时更新文件的内容即可

写一个`get_available_mem.sh`脚本来提供文件内容

```
#!/bin/sh
#
# * * * * * zoker /bin/bash /home/zoker/app/node_exporter/get_available_mem.sh > /home/zoker/app/node_exporter/get_available_mem.prom
#
echo "# HELP get_available_mem get available mem test case"
echo "# TYPE get_available_mem gauge"
freeme=`top -bn 1 -i -c| head -n 5| tail -n 1| awk '{print $9}' | sed 's/.*\(...\)$/\1/'`
echo "get_available_me $freeme"
```

这里我们为了能够展现出变化，通过`sed`命令取了后三位，配置到`crontab`：
```
vi /etc/crontab
* * * * * zoker /bin/bash /home/zoker/app/node_exporter/get_available_mem.sh > /home/zoker/app/node_exporter/get_available_mem.prom
```
这样就会定时的更新`get_available_mem.prom`文件的内容了，这个时候我们来配置 Grafana 的显示

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0902/184945_ae6046a7.png "在这里输入图片标题")

可以看到刚刚添加的指标会自动出现在 Metrics 的选项里，选择即可看到图形的变化。

### 二、编写`expoter`收集随机数并展示

这里以`Golang`为例，使用 Prometheus 提供的 Golang SDK 编写，下面是一个使用包里面提供的自有 Handler 来暴露一些基本的 Golang 信息的 Exporter:

```
package main

import (
    "flag"
    "log"
    "net/http"

    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var addr = flag.String("listen-address", ":9192", "The address to listen on for HTTP requests.")

func main() {
    flag.Parse()
    http.Handle("/metrics", promhttp.Handler())
    log.Fatal(http.ListenAndServe(*addr, nil))
}
```

执行`go mod init main`和`go build self_exporter.go`并执行，访问`http://127.0.0.1:9192/metrics`可看到如下信息
```
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 4.7424e-05
go_gc_duration_seconds{quantile="0.25"} 4.7424e-05
go_gc_duration_seconds{quantile="0.5"} 4.7424e-05
go_gc_duration_seconds{quantile="0.75"} 4.7424e-05
go_gc_duration_seconds{quantile="1"} 4.7424e-05
go_gc_duration_seconds_sum 4.7424e-05
go_gc_duration_seconds_count 1
# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
go_goroutines 8
....
```
是不是感觉很熟悉，上面我们使用`textfile`方式自定义指标的时候也是这样的格式，这里还只是默认的一些指标，下面我们添加一个 Gauge 类型的指标，提供一些随机的数值

```
package main

import (
    "flag"
    "log"
    "net/http"
    "math/rand"
    "time"

    "github.com/prometheus/client_golang/prometheus/promhttp"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus"
)

var addr = flag.String("listen-address", ":9192", "The address to listen on for HTTP requests.")

func main() {
    flag.Parse()
    addRandNumbers() //调用函数
    http.Handle("/metrics", promhttp.Handler())
    log.Fatal(http.ListenAndServe(*addr, nil))
}

func addRandNumbers() { // 起一个协程用来定时更新这个 randGauge
    go func() {
        for {
            randGauge.Set(rand.Float64() * 100)
            time.Sleep(2 * time.Second)
        }
    }()
}

var ( // 定义一个 Gauge 类型的可变指标
    randGauge = promauto.NewGauge(prometheus.GaugeOpts{
        Name: "test_random_numbers",
        Help: "test random numbers",
    })
)
```

重新编译运行，可以看到已经有相关数值了

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0902/210805_66709dd8.png "在这里输入图片标题")

我们把这个自定义的`exporter`配置为 Prometheus 的一个实例

```
  - job_name: 'jmaster'
    static_configs:
    - targets: ['192.168.31.180:9100', '192.168.31.180:9192']
      labels:
        instance: jmaster
```
不要忘记重启 Prometheus ,我们进入到 Grafana 即可在 Metrics 看到我们刚刚添加的随机数指标（真是一对好CP）

![输入图片说明](https://blogine-1251619080.cos.ap-guangzhou.myqcloud.com/uploads/images/2020/0902/213757_04ca4bfc.png "在这里输入图片标题")

##### 总结

通过以上两种方式我们不仅可以使用 Grafana 来进行监控，还可以进行一些数据的展示，毕竟 Grafana 本身就是一个数据度量展示的平台，比如：

- 注册用户数、创建仓库数量、网站浏览量等
- Jenkins 构建任务成功失败数（[有插件了](http://https://plugins.jenkins.io/prometheus/)）
- 访客数量、评论数量
- 个人推送数据、代码量、工作效能度量展示
- .....