---
layout:	post
title:	"基于Zookeeper的服务注册与发现"
date:	2015-12-25 12:25:12
categories: zookeeper
excerpt: 基于Zookeeper的服务注册与发现的架构
---

* content
{:toc}

## 背景

大多数系统都是从一个单一系统开始起步的，随着公司业务的快速发展，这个单一系统变得越来越庞大，带来几个问题：

1. 随着访问量的不断攀升，纯粹通过提升机器的性能来已经不能解决问题，系统无法进行有效的水平扩展
2. 维护这个单一系统，变得越来越复杂
3. 同时，随着业务场景的不同以及大研发的招兵买马带来了不同技术背景的工程师，在原有达达Python技术栈的基础上，引入了Java技术栈。

如何来解决这些问题?业务服务化是个有效的手段来解决大规模系统的性能瓶颈和复杂性。通过系统拆分将原有的单一庞大系统拆分成小系统，它带来了如下好处：

1. 原来系统的压力得到很好的分流，有效地解决了原先系统的瓶颈，同时带来了更好的扩展性
2. 独立的代码库，更少的业务逻辑，系统的维护性得到极大的增强

同时，也带来了一系列问题：

* 随着系统服务的越来越多，如何来管理这些服务?
* 如何分发请求到提供同一服务的多台主机上(负载均衡如何来做)
* 如果提供服务的Endpoint发生变化，如何将这些信息通知服务的调用方?

---

### 最初的解决方案

Linkedin的创始人里德霍夫曼曾经说过:

成立一家初创公司就像把自己从悬崖上扔下来，在降落过程中去组装一架飞机。

这对于初创公司达达也是一样的，业务在以火箭般的速度发展着。技术在业务发展中作用就是保障业务的稳定运行，快速地“组装一架飞机”。所以，在业务服务化的早期，我们采用了Nginx+本地hosts文件的方式进行内部服务的注册与发现，架构图如下：

![dada-framework]({{"/css/pics/dada-services-framework.jpg"}})

各系统组件角色如下：

1. 服务消费者通过本地hosts中的服务提供者域名与Nginx的IP绑定信息来调用服务
2. Nginx用来对服务提供者提供的服务进行存活检查和负载均衡
3. 服务提供者提供服务给服务消费者访问，并通过Nginx来进行请求分发

这在内部系统比较少，访问量比较小的情况下，解决了服务的注册，发现与负载均衡等问题。但是，随着内部服务越来愈多，访问量越来越大的情况下，该架构的隐患逐渐暴露出来：

* 最明显的问题是Nginx存在单点故障(SPOF)，同时随着访问量的提升，会成为一个性能瓶颈
* 随着内部服务的越来越多，不同的服务消费方需要配置不同的hosts，很容易在增加新的主机时忘记配置hosts导致服务不能调用问题，增加了运维负担
* 服务的配置信息分散在各个主机hosts中，难以保持一致性，不便于服务的管理
* 服务主机的发布和下线需要手工的修改Nginx upstream配置，修改的配置需要上线，不利于服务的快速部署

---

### 如何来解决

在谈如何来解决之前，现梳理一下服务注册与发现的目标：

* 服务的注册信息应该统一保存，方便于服务的管理
* 自动通过服务的名称去发现服务，而不必了解这个服务提供的end-point到底是哪台主机
* 支持服务的负载均衡及fail-over
* 增加或移除某个服务的end-point时，对于服务的消费者来说是透明的
* 支持Python和Java

---

#### 备选方案一: DNS

DNS作为服务注册发现的一种方案，它比较简单。只要在DNS服务上，配置一个DNS名称与IP对应关系即可。定位一个服务只需要连接到DNS服务器上，随机返回一个IP地址即可。由于存在DNS缓存，所以DNS服务器本身不会成为一个瓶颈。

这种基于Pull的方式不能及时获取服务的状态的更新(例如：服务的IP更新等)。如果服务的提供者出现故障，由于DNS缓存的存在，服务的调用方会仍然将请求转发给出现故障的服务提供方;反之亦然。

---

#### 备选方案二：Dubbo

Dubbo是阿里巴巴推出的分布式服务框架，致力于解决服务的注册与发现，编排，治理。它的优点如下：

1. 功能全面，易于扩展
2. 支持各种串行化协议(JSON，Hession，java串行化等)
3. 支持各种RPC协议(HTTP，Java RMI，Dubbo自身的RPC协议等)
4. 支持多种负载均衡算法
5. 其他高级特性：服务编排，服务治理，服务监控等

缺点如下：

1. 只支持Java，对于Python没有相应的支持
2. 虽然已经开源，但是没有成熟的社区来运营和维护，未来升级可能是个麻烦
3. 重量级的解决方案带来新的复杂性

---

#### 备选方案三：Zookeeper

Zookeeper是什么?按照Apache官网的描述是：

ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.

参照官网的定义，它能够做：

1. 作为配置信息的存储的中心服务器
2. 命名服务
3. 分布式的协调
4. Mater选举等

在定义中特别提到了命名服务。在调研之后，Zookeeper作为服务注册与发现的解决方案，它有如下优点：

1. 它提供的简单API
2. 已有互联网公司(例如：Pinterest，Airbnb)使用它来进行服务注册与发现
3. 支持多语言的客户端
4. 通过Watcher机制实现Push模型，服务注册信息的变更能够及时通知服务消费方

缺点是：

1. 引入新的Zookeeper组件，带来新的复杂性和运维问题
2. 需自己通过它提供的API来实现服务注册与发现逻辑(包含Python与Java版本)

我们对上述几个方案的优缺点权衡之后，决定采用了基于Zookeeper实现自己的服务注册与发现。

---

### 基于Zookeeper的服务注册与发现架构

![zookeeper-register-locate]({{"/css/pics/zookeeper-services-register-locate.jpg"}})

在此架构中有三类角色：服务提供者，服务注册中心，服务消费者。

服务提供者

服务提供者作为服务的提供方将自身的服务信息注册到服务注册中心中。服务信息包含：

* 隶属于哪个系统
* 服务的IP，端口
* 服务的请求URL
* 服务的权重等等

服务注册中心

服务注册中心主要提供所有服务注册信息的中心存储，同时负责将服务注册信息的更新通知实时的Push给服务消费者(主要是通过Zookeeper的Watcher机制来实现的)。

服务消费者

服务消费者主要职责如下：

1. 服务消费者在启动时从服务注册中心获取需要的服务注册信息
2. 将服务注册信息缓存在本地
3. 监听服务注册信息的变更，如接收到服务注册中心的服务变更通知，则在本地缓存中更新服务的注册信息
4. 根据本地缓存中的服务注册信息构建服务调用请求，并根据负载均衡策略(随机负载均衡，Round-Robin负载均衡等)来转发请求
5. 对服务提供方的存活进行检测，如果出现服务不可用的服务提供方，将从本地缓存中剔除

服务消费者只在自己初始化以及服务变更时会依赖服务注册中心，在此阶段的单点故障通过Zookeeper集群来进行保障。在整个服务调用过程中，服务消费者不依赖于任何第三方服务。

---

#### 实现机制介绍

Zookeeper数据模型介绍

在整个服务注册与发现的设计中，最重要是如何来存储服务的注册信息。

在设计基于Zookeeper的服务注册结构之前，我们先来看一下Zookeeper的数据模型。Zookeeper的数据模型如下图所示：

![zookeeper-data-model]({{"/css/pics/zookeeper-data-model.jpg"}})

Zookeeper数据模型结构与Unix文件系统很类似，是一个树状层次结构。每个节点叫做Znode，节点可以拥有子节点，同时允许将少量数据存储在该节点下。客户端可以通过监听节点的数据变更和子节点变更来实时获取Znode的变更(Wather机制)。

#### 服务注册结构

![zookeeper-register-agency]({{"/css/pics/zookeeper-services-register-agency.jpg"}})

服务注册结构如上图所示。

* /dada来标示公司名称dada，同时能方便与其它应用的目录区分开(例如：Kafka的brokers注册信息放置在/brokers下)
* /dada/services将所有服务提供者都放置该目录下
* /dada/services/category1目录定义具体的服务提供者的id：category1，同时该Znode节点中允许存放该服务提供者的一些元数据信息，例如：名称，服务提供者的Owner，上下文路径(Java Web项目),健康检查路径等。该信息可以根据实际需要进行自由扩展。
* /dada/services/category1/helloworld节点定义了服务提供者category1下的一个服务：helloworld。其中helloworld为该服务的ID，同时允许将该服务的元数据信息存储在该Znode下，例如图中标示的：服务名称，服务描述，服务路径，服务的调用的schema，服务的调用的HTTP METHOD等。该信息可以根据实际需要进行自由扩展。
* /dada/services/category1/helloworld/providers节点定义了服务提供者的父节点。在这里其实可以将服务提供者的IP和端口直接放置在helloworld节点下，在这里单独放一个节点，是为了将来可以将服务消费者的消息挂载在helloworld节点下，进行一些扩展，例如命名为：/dada/services/category1/helloworld/consumers。
* /dada/services/category__1/helloworld/providers/192.168.1.1:8080该节点定义了服务提供者的IP和端口，同时在节点中定义了该服务提供者的权重。

#### 实现机制

由于目前服务注册通过我们的服务注册中心UI来进行注册，这部分逻辑比较简单，即通过UI界面来构造上述定义的服务注册结构。

下面着重介绍一下我们的服务发现是如何工作的：

![zookeeper-services-locate]({{"/css/pics/zookeeper-services-locate.jpg"}})

在上述类图中，类ServiceDiscovery主要通过Zookeeper的API(Python/Java版本)来获取服务信息，同时对服务注册结构中的每个服务的providers节点增加Watcher，来监控节点变化。获取的服务注册信息保存在变量service_repos中。通过在初始化时设置LoadBalanceStrategy的实现(Round-Robin算法，Radmon算法)来实现服务提供者的负载均衡。主要方法:

1. init获取Zookeeper的服务注册信息，并缓存在service_repos
2. get_service_repos方法获取实例变量service_repos
3. get_service_endpoint根据init构建好的service_repos，以及lb_strategy提供的负载均衡策略返回某个服务的URL地址
4. update_service_repos通过Zookeeper的Watcher机制来实时更新本地缓存service_repos
5. heartbeat_monitor是一个心跳检测线程，用来进行服务提供者的健康存活检测，如果出现问题，将该服务提供者从该服务的提供者列表中移除;反之，则加入到服务的提供者列表中

LoadBalanceStrategy定义了根据服务提供者的信息返回对应的服务Host和IP，即决定由那台主机+端口来提供服务。

RoundRobinStrategy和RandomStrategy分别实现了Round-Robin和随机的负载均衡算法

---

### 未来展望

目前达达基于Zookeeper的服务注册与发现的架构还处于初期，很多功能还未完善，例如：服务的路由功能，与部署平台的集成，服务的监控等等。

当然基于Zookeeper还能做其它许多事情，例如：实时动态配置系统。目前，我们已经基于Zookeeper实现了实时动态配置系统。

[原文链接](http://www.techweb.com.cn/network/hardware/2015-12-25/2246973.shtml)