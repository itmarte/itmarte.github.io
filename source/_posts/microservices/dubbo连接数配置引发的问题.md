---
title: dubbo 连接数配置导致连接爆发
tags:
  - 分布式与微服务
  - dubbo 配置
categories:
  - 分布式与微服务
  - dubbo配置
date: 2020-06-04 15:31:13
---

## 0x00. 翻车现场

收到运维noc告警：

![img](/images/0082zybply1gbqenpjnt7j30yq0higpo.jpg)

## 0x01. 历尽艰辛，深入排查

打开电脑，首先确认生产交易一切还正常。查看这段时间日志，发现并没有什么异常情况，日志都是正常输出。没办法只好再次走查此次改动的代码，发现全是业务代码，并没有任何与网络连接有关的代码改动。

`netstat -anp|grep 6701|wc -l `  查看连接情况

![img](/images/5911899802886.png)



`jmap -histo 6701|less` 查看堆内存情况

![020060416292](/images/20200604162924.png)

问题真的请奇怪，一时半会想不到解决方案，只好先实施重启大法。重启过后，连接数下降了，到达了正常阈值。但是不一会连接数持续升高，不一会还是升到上万。

这下重启解决不了办法，只好从应用出发，找找到底什么问题。

这个应用是一个路由服务，会根据上游系统指定路由编码，将交易分发到下游子系统。架构图如下:

![img](/images/0082zybply1gbr8va8yvhj30hm0ck3z8.jpg)

之前在这篇文章[路由系统演化史](https://mp.weixin.qq.com/s/Det95SU1u1dDH7nT_B1XEQ)讲过，路由系统使用 **Dubbo API** ，代码如下：

![img](/images/0082zybply1gbqenooa0ej314i0u0tf3.jpg)

由于我们还有另外一套系统，也部署这个应用，但是该系统生产机器连接数却很少。交叉比对了两套系统应用的系统配置值，只有 **connections** 设置不一样，当前有问题的系统设置为 **1000**，另外一个系统为 **10** 。

大致找到原因，也将 **connections** 设置为 **10**，重启应用，生产机器连接数恢复正常。

## 0x02. 抽丝剥茧，还原经过

首先我们来看下 **connections** 这个配置的作用，可以直接查看官方文档<http://dubbo.apache.org/zh-cn/docs/user/references/xml/dubbo-reference.html>。

下面配置来源于：**dubbo:reference**

![img](/images/0082zybply1gbpy5n5wfuj31m6078wp5.jpg)

总共可以在三个地方配置 **connections** 参数，分别为：**dubbo:reference**，**dubbo:consumer**，**dubbo:provider**。

> **注意**：图中标示地方实际上与源码存在出入。截止 **Dubbo 2.7.3** 版本，图中 ① 处，**dubbo:consumer** 文档上显示为 **100**，实际源码默认配置为 **0**，这点需要注意。另外 ② 处文字描述存在问题，目前 **connections** 参数主要对 **dubbo** 协议有用，**http** 短连接协议还未使用该配置

其中 **reference.connections** 为服务级别的配置，若未配置将会使用 **consumer.connections** 配置值。另外这个参数若在 **provider.connections** 配置，其对服务提供者无效，参数将通过注册中心传递给消费者成为其默认配置。三者实际作用顺序如下：

![img](/images/0082zybply1gbpxzrh8q4j31mc0u0hdt.jpg)

**Debug** 源码，**connections** 最终会在 **DubboProtocol#getClients** 被使用，方法源码如下：

![img](/images/0082zybply1gbqenq2xc6j31820cs76u.jpg)

![img](/images/0082zybply1gbqenma2pxj31au0u044x.jpg)

> **Dubbo** 协议默认将会使用 **Netty** 与服务提供者建立长连接

首先将会获取 **connections** 配置，规则如上图，若其大于 **0**，建立 **connections** 数量的长连接。

如果一个提供者对外暴露 **10** 个接口，且其有两个节点。消费者端引入提供者所有服务，配置 **connections=1000**。当消费者启动之后，将会立刻创建 **1000x2x10=20000** 连接。**这就是生产机器连接数飙升的根本原因**。

> 路由服务使用 **Dubbo API** 编程，服务启动成功之后，只有上游系统调用路由服务时， **Dubbo** 才会与与下游服务提供者建立连接，所以现象看起来服务连接数是慢慢激增。

如果未设置 **connections** 参数，Dubbo 将会创建**共享连接（shareconnections）**。消费者调用的服务若为同一个服务提供者（**IP+PORT** 区分），这些服务接口将会共享这些连接。

**shareconnections** 可以在 **dubbo:consumer** 配置中配置，也可以在启动 **JVM** 参数加入如下配置：

```
-Dshareconnections=10
```

如果消费者需要调用同个服务提供者应用的 **10** 个服务接口，服务提供者提供两个节点，**shareconnections=1000**，消费者服务启动之后，仅会创建 **1000\*2=2000** 连接。

这么对比，**shareconnections** 与 **connections** 建立连不是一个量级。

### 2.1 使用连接

消费者调用服务时，将会随机从连接数组中取一个连接使用，代码位于 `DubboInvoker#doInvoke`。

![img](/images/0082zybply1gbqenjpogfj30ph05zdgv.jpg)

### 2.2 如何正确配置连接数

首先我们来看下单一长连接性能，文档地址:<http://dubbo.apache.org/zh-cn/docs/user/references/protocol/dubbo.html>

![img](/images/0082zybply1gbq3md7d15j30m90cb43g.jpg)

对于只有少数消费者场景，我们可以使用默认配置，即不配置 **connections** 参数 。若调用同一个提供者服务过多，可以考虑适当多配增加 **shareconnections**。最后若某一服务接口调用量特别大，可以考虑为这个服务单独配置 **connections**。

## 0x03. 举一反三，聊聊其他配置

Dubbo 还有很多配置项，下面着重介绍一些配置参数。

### 3.1 dubbo.provider.executes

该参数用来控制每个方法最大并行数。如果该值设置为 **10** ，每个服务方法若已有 **10** 个请求正在处理，第 **11** 个服务请求将会抛出异常，直到之前服务调用完成，正在请求数量小于 **10** 未知。

![img](/images/0082zybply1gbq3z7a7mmj31nq074qat.jpg)

一旦设置 **executes>0**,**Dubbo** 将会通过 **SPI** 机制启用 `ExecuteLimitFilter`，源码还是比较简单。

![img](/images/0082zybply1gbqennntwoj30sg0fswhn.jpg)

### 3.2 dubbo.reference.actives

这个参数将会控制消费者每个服务每个方法最大并发数。可以通过 **dubbo:method.actives** 单独为服务方法设置。如果该值为 **10**，一旦某个服务某个方法并发数超过 **10**，第 **11** 个服务将会等待，若在超时时间内其他请求执行结束，计数值减值小于阈值，第 **11** 个请求将会被执行，否者将会抛错。

![img](https://tva1.sinaimg.cn/large/0082zybply1gbr8wael1dj31lw07qdgq.jpg)

> `dubbo.provider` 上也可以配置这个值，其将会与 **connections** 一样，将会传递给消费者。

原理等同上面方法，将会启用 `ActiveLimitFilter`，源码如下 ：

![img](/images/0082zybply1gbqenluvenj30rz0hsjv2.jpg)

这里需要注意 **actives** 引起超时与服务端超时区别。

![img](/images/0082zybply1gbqenlanqrj327f0u04gd.jpg)

### 3.3 dubbo.protocol.accepts

服务提供者最大连接数，如果设置 **accepts=10**,一旦服务提供者连接数大于 **10**，其余新增连接将会被拒绝。

![img](/images/0082zybply1gbq4mytke2j31gw07279o.jpg)

方法源码如下：

![img](/images/0082zybply1gbqemv4o7qj32780lkgrc.jpg)

服务提供者断开连接，消费端将会打印连接断开日志。另外消费者会定时检查长连接可用性，若不可用，将会重新发起连接。所以在消费者端就会看到连接断开，重连，然后又被服务提供者断开的现象。

![img](/images/0082zybply1gbqenms1eqj32dw0l87h5.jpg)

## 0x04. 总结

本文通过一次生产连接数过多的现象，详细剖析定位问题的原因。作为一个合格的开发，对于开源框架，我们不仅要会熟练使用，也要了解其底层实现，相关参数设置。一旦参数设置不合理就可能引发生产事故。

另外对于生产系统，监控系统非常重要。比如上面的问题，如果没有监控发现，小黑哥可能一时半会都不知道有这个问题存在，毕竟平时也不会太关注连接数这个指标。