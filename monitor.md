# 利用Influxdb+Grafana搭建自己的业务监控系统

### 为什么要选择Influxdb+Grafana

先说说我们的场景和需求。我们的系统是一个对接了多家数据上游和下游的中间平台，由于上游系统的能力参差不齐，经常会出现服务异常情况，为了将各个数据源的运行情况做一个统一的把控，并且在异常情况下能有一些应急处理措施，我们需要对业务系统做一些监控。总的看来，我们的需求有下面几个特点：

1. 纯业务监控，高度定制化。相比运维层面的监控而言，我们的监控更多的是业务层面，需要自己定义监控项，不涉及机器负载、磁盘、网络等方面。
2. 实时监控。要想对突发的异常情况做紧急处理，监控就必须得是（准）实时的。这对监控系统的性能要求比较高，不光要求响应要快，而且要应付随时可能的大流量冲击。
3. 简单。包括安装，使用，运维等，毕竟是自运维的，不是专业运维出身，太复杂的系统操作成本太高了。对于指标的定义上也是简单清晰，能有图形展示即可。

其他的特点诸如指标分析，图形展示等以后再补充。

起初我也调研过其他的监控产品，如[Graphite](http://graphiteapp.org/)，[open-falcon](http://open-falcon.org/)。这两者都有比较强大的能力，提供了非常丰富的监控指标和图形化展示界面，但是都太偏运维层面，部署起来也不太方便。

说说Influxdb和Grafana的优势吧。首先Influxdb，比较轻量简单的，用Go编写，性能也足够好，部署相当方便（使用standalone的binary甚至都不需要安装），最主要的是，原生就支持HTTP请求，操作起来相当方便。Grafana部署起来稍微麻烦一点，但是扩展性极好，有着丰富的插件及数据源支持，并且支持图形化的配置界面，使用起来也很方便。下面来说说安装流程吧。

### Install

#### Grafana

我的系统是centos，在centos下可以直接通过命令

```shell
sudo yum install https://grafanarel.s3.amazonaws.com/builds/grafana-3.1.1-1470047149.x86_64.rpm
```

进行安装。

启动Grafana的话需要到安装目录下，启动的示例脚本如下：

```shell
cd /usr/share/grafana && /usr/sbin/grafana-server --pidfile=/home/chen/run/grafana-server.pid --config=/etc/grafana/grafana.ini cfg:default.paths.data=/home/chen/grafana/lib cfg:default.paths.logs=/home/chen/grafana/log cfg:default.paths.plugins=/home/chen/grafana/lib/plugins
```

启动完成之后，就可以在浏览器中打开http://host:3000链接进行配置了。

#### InfluxDB

InfluxDB的官网地址：[https://www.influxdata.com/downloads/#influxdb](https://www.influxdata.com/downloads/#influxdb)

centos安装[influxdb](https://github.com/influxdata/influxdb) 可以使用以下命令：

```shell
wget https://dl.influxdata.com/influxdb/releases/influxdb-1.0.0.x86_64.rpmsudo yum localinstall influxdb-1.0.0.x86_64.rpm
```

也可以下载standalone的二进制文件，下载即可用。

### 配置

#### Grafana

Grafana没有太多可说的，图形化界面操作，比较简单。

#### InfluxDB

influxdb的配置都在influxdb.conf配置文件中。下面的命令以指定的配置文件运行server：

```shell
influxd -config influxdb.conf
```

默认tcp监听端口在8088，如果要进行变动的话，需要在influxdb.conf中添加一行：

```xml
bind-address = "<IP>:8000"
```

### 写入数据

主要参考Influxdb的[Document](https://docs.influxdata.com/influxdb/v1.0/introduction/getting_started/)。

influxDB操作写入数据的格式如下：

> <measurement>[,<tag-key>=<tag-value>...] <field-key>=<field-value>[,<field2-key>=<field2-value>...][unix-nano-timestamp]

通过HTTP接口写入数据的示例如下：

```shell
curl -XPOST 'http://localhost:8086/write?db=mydb' -d 'cpu,host=server01,region=uswest load=42 1434055562000000000'
```
