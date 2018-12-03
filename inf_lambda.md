原文链接：[大数据系统的Lambda架构](https://zhuanlan.zhihu.com/p/20510974)

Nathan Marz的大作_Big Data: Principles and best practices of scalable real-time data systems_介绍了Labmda Architecture的概念，用于在大数据架构中，如何让real-time与batch job更好地结合起来，以达成对大数据的实时处理。

### 传统系统的问题

在传统数据库的设计中，无法很好地支持系统的可伸缩性。当用户访问量增加时，数据库无法满足日益增长的用户请求负载，从而导致数据库服务器无法及时响应用户请求，出现超时错误。

解决的办法是在Web服务器与数据库之间增加一个异步处理的队列。如下图所示：

![img](https://pic3.zhimg.com/80/304eeaf999918c1fd08d9a3b869184a6_hd.png)

当Web Server收到页面请求时，会将消息添加到队列中。在DB端，创建一个Worker定期从队列中取出消息进行处理，例如每次读取100条消息。这相当于在两者之间建立了一个缓冲。

但是，这一方案并没有从本质上解决数据库overload的问题，且当worker无法跟上writer的请求时，就需要增加多个worker并发执行，数据库又将再次成为响应请求的瓶颈。一个解决办法是对数据库进行分区（horizontal partitioning或者sharding）。分区的方式通常以Hash值作为key。这样就需要应用程序端知道如何去寻找每个key所在的分区。

问题仍然会随着用户请求的增加接踵而来。当之前的分区无法满足负载时，就需要增加更多分区，这时就需要对数据库进行reshard。resharding的工作非常耗时而痛苦，因为需要协调很多工作，例如数据的迁移、更新客户端访问的分区地址，更新应用程序代码。如果系统本身还提供了在线访问服务，对运维的要求就更高。稍有不慎，就可能导致数据写到错误的分区，因此必须要编写脚本来自动完成，且需要充分的测试。

即使分区能够解决数据库负载问题，却还存在容错性（Fault-Tolerance）的问题。解决办法：

- 改变queue/worker的实现。当消息发送给不可用的分区时，将消息放到“pending”队列，然后每隔一段时间对pending队列中的消息进行处理。
- 使用数据库的replication功能，为每个分区增加slave。

问题并没有得到完美地解决。假设系统出现问题，例如在应用系统代码端不小心引入了一个bug，使得对页面的请求重复提交了一次，这就导致了重复的请求数据。糟糕的是，直到24小时之后才发现了该问题，此时对数据的破坏已经造成了。即使每周的数据备份也无法解决此问题，因为它不知道到底是哪些数据受到了破坏（corrupiton）。由于人为错误总是不可避免的，我们在架构时应该如何规避此问题？

现在，架构变得越来越复杂，增加了队列、分区、复制、重分区脚本（resharding scripts）。应用程序还需要了解数据库的schema，并能访问到正确的分区。问题在于：数据库对于分区是不了解的，无法帮助你应对分区、复制与分布式查询。最糟糕的问题是系统并没有为人为错误进行工程设计，仅靠备份是不能治本的。归根结底，系统还需要限制因为人为错误导致的破坏。

### 数据系统的概念

大数据处理技术需要解决这种可伸缩性与复杂性。首先要认识到这种分布式的本质，要很好地处理分区与复制，不会导致错误分区引起查询失败，而是要将这些逻辑内化到数据库中。当需要扩展系统时，可以非常方便地增加节点，系统也能够针对新节点进行rebalance。

其次是要让数据成为不可变的。原始数据永远都不能被修改，这样即使犯了错误，写了错误数据，原来好的数据并不会受到破坏。

何谓“数据系统”？Nathan Marz认为：

> 如果数据系统通过查找过去的数据去回答问题，则通常需要访问整个数据集。

因此可以给data system的最通用的定义：

```text
Query = function(all data)
```

接下来，本书作者介绍了Big Data System所需具备的属性：

- 健壮性和容错性（Robustness和Fault Tolerance）
- 低延迟的读与更新（Low Latency reads and updates）
- 可伸缩性（Scalability）
- 通用性（Generalization）
- 可扩展性（Extensibility）
- 内置查询（Ad hoc queries）
- 维护最小（Minimal maintenance）
- 可调试性（Debuggability）

### Lambda架构

Lambda架构的主要思想就是将大数据系统构建为多个层次，如下图所示：

![img](https://pic1.zhimg.com/80/7c41d5f9f76376ecc1faff48f0a259fc_hd.png)

理想状态下，任何数据访问都可以从表达式Query = function(all data)开始，但是，若数据达到相当大的一个级别（例如PB），且还需要支持实时查询时，就需要耗费非常庞大的资源。

一个解决方式是预运算查询函数（precomputed query funciton）。书中将这种预运算查询函数称之为**Batch View**，这样当需要执行查询时，可以从Batch View中读取结果。这样一个预先运算好的View是可以建立索引的，因而可以支持随机读取。于是系统就变成：

```text
batch view = function(all data)
query = function(batch view)
```

#### Batch Layer

在Lambda架构中，实现batch view = function(all data)的部分被称之为**batch layer**。它承担了两个职责：

- 存储Master Dataset，这是一个不变的持续增长的数据集
- 针对这个Master Dataset进行预运算

显然，Batch Layer执行的是批量处理，例如Hadoop或者Spark支持的Map-Reduce方式。 它的执行方式可以用一段伪代码来表示：

```text
function runBatchLayer():
  while (true):
    recomputeBatchViews()
```

例如这样一段代码：

```text
Api.execute(Api.hfsSeqfile("/tmp/pageview-counts"),
     new Subquery("?url", "?count")
         .predicate(Api.hfsSeqfile("/data/pageviews"),
             "?url", "?user", "?timestamp")
         .predicate(new Count(), "?count");
```

代码并行地对hdfs文件夹下的page views进行统计（count），合并结果，并将最终结果保存在pageview-counts文件夹下。

利用Batch Layer进行预运算的作用实际上就是将大数据变小，从而有效地利用资源，改善实时查询的性能。但这里有一个前提，就是我们需要预先知道查询需要的数据，如此才能在Batch Layer中安排执行计划，定期对数据进行批量处理。此外，还要求这些预运算的统计数据是支持合并（merge）的。

#### Serving Layer

Batch Layer通过对master dataset执行查询获得了batch view，而Serving Layer就要负责对batch view进行操作，从而为最终的实时查询提供支撑。因此Serving Layer的职责包含：

- 对batch view的随机访问
- 更新batch view

Serving Layer应该是一个专用的分布式数据库，例如Elephant DB，以支持对batch view的加载、随机读取以及更新。注意，它并不支持对batch view的随机写，因为随机写会为数据库引来许多复杂性。简单的特性才能使系统变得更健壮、可预测、易配置，也易于运维。

#### Speed Layer

只要batch layer完成对batch view的预计算，serving layer就会对其进行更新。这意味着在运行预计算时进入的数据不会马上呈现到batch view中。这对于要求完全实时的数据系统而言是不能接受的。要解决这个问题，就要通过speed layer。从对数据的处理来看，speed layer与batch layer非常相似，它们之间最大的区别是前者只处理最近的数据，后者则要处理所有的数据。另一个区别是为了满足最小的延迟，speed layer并不会在同一时间读取所有的新数据，相反，它会在接收到新数据时，更新realtime view，而不会像batch layer那样重新运算整个view。speed layer是一种增量的计算，而非重新运算（recomputation）。

因而，Speed Layer的作用包括：

- 对更新到serving layer带来的高延迟的一种补充
- 快速、增量的算法
- 最终Batch Layer会覆盖speed layer

Speed Layer的等式表达如下所示：

```text
realtime view = function(realtime view, new data)
```

注意，realtime view是基于新数据和已有的realtime view。

总结下来，Lambda架构就是如下的三个等式：

```text
batch view = function(all data)
realtime view = function(realtime view, new data)
query = function(batch view . realtime view)
```

整个Lambda架构如下图所示：

![img](https://pic4.zhimg.com/80/3f93b809110b83a8f034d83daefe8a2f_hd.png)

基于Lambda架构，一旦数据通过batch layer进入到serving layer，在realtime view中的相应结果就不再需要了。
