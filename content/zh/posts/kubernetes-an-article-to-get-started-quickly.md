---
title: 'Kubernetes一篇快速入门的文章'
ShowRssButtonInSectionTermList: true
cover.image:
date: 2022-04-28T23:38:11+08:00
draft: false
showtoc: true
tocopen: true
type: posts
author: ["Xinwei Xiong", "Me"]
keywords: ["Kubernetes", "k8s", "Docker", Cloud Native", "CNCF"]
tags:
  - blog
  - kubernetes
  - k8s
categories:
  - Development
  - Blog
  - Kubernetes
description: >
    Kubernetes是一个开源的容器编排引擎，用于自动化部署、扩展和管理容器化应用程序。该项目由云原生计算基金会管理，该基金会由Linux基金会托管。这篇文章将会带你快速入门Kubernetes。
---


## 正片开始~

Kubernetes 是 Google 团队发起的一个开源项目，它的目标是管理跨多个主机的容器，用于自动部署、扩展和管理容器化的应用程序，主要实现语言为 Go 语言。Kubernetes 的组件和架构还是相对较复杂的。要慢慢学习~

> 我们急需编排一个容器~



## 为什么 kubernetes 弃用了 docker

::: tip 很意外
听到 Kubernetes 从 Kubernetes 版本 1.20 开始弃用对 Docker 作为容器运行时的支持，这似乎有点令人震惊。

Kubernetes 正在删除对 Docker 作为**容器运行时**的支持。Kubernetes 实际上并不处理在机器上运行容器的过程。相反，它依赖于另一个称为**容器运行时**的软件。.

:::

docker 比 kubernetes 发行的早

docker本身不兼容 CRI 接口。Kubernetes 适用于所有实现称为容器运行时**接口 （CRI） 标准的容器运行时**。这本质上是 Kubernetes 和容器运行时之间通信的标准方式，任何支持此标准的运行时都会自动与 Kubernetes 配合使用。

Docker 不实现容器运行时接口 （CRI）。过去，容器运行时没有那么多好的选择，Kubernetes 实现了 Docker shim，这是一个额外的层，用作 Kubernetes 和 Docker 之间的接口。然而，现在有很多运行时可以实现 CRI，Kubernetes 维护对 Docker 的特殊支持不再有意义。



::: warning 弃用的意义
虽然移除了 docker ，但是还是保留了以前的 dockershim，如果你愿意，你依旧可以使用 docker 容器化引擎提供容器化支持。

除了docker，还有 containerd、CRI-O

我会告诉你一个秘密：**Docker实际上不是一个容器运行时**！它实际上是一个工具集合，位于称为**containerd**的容器运行时之上。.

没错！Docker 不直接运行容器。它只是在单独的底层容器运行时之上创建一个更易于访问且功能更丰富的界面。当它被用作 Kubernetes 的容器运行时时，Docker 只是 Kubernetes 和容器之间的中间人。

但是，Kubernetes可以直接使用containerd作为容器运行时，这意味着在这个中间人角色中不再需要Docker。Docker仍然有很多东西可以提供，即使在Kubernetes生态系统中也是如此。只是不需要专门用作容器运行时。

:::



**Podman 横空出世：**

podman 的定位也是与 docker 兼容，因此也在使用上尽量接近 docker。在使用方面，可以分为两个方面，一是系统构建者的角度，二是系统使用者的角度。



## kubernetes(k8s)

[Kubernetes](http://kubernetes.io/)是Google基于Borg开源的容器编排调度引擎，作为[CNCF](http://cncf.io/)（Cloud Native Computing Foundation）最重要的组件之一，它的目标不仅仅是一个编排系统，而是提供一个规范，可以让你来描述集群的架构，定义服务的最终状态，`Kubernetes` 可以帮你将系统自动地达到和维持在这个状态。`Kubernetes` 作为云原生应用的基石，相当于一个云操作系统，其重要性不言而喻。

> **一句话介绍说：k8s就是为我们提供了可弹性运行分布式系统框架，k8s满足我的扩展要求、故障转移、部署模式等等，例如：k8s可以轻松管理系统的Canary部署。**



::: tip sealos 是什么
**[sealos](https://www.sealos.io/zh-Hans/docs/Intro) 是以 kubernetes 为内核的云操作系统发行版**

早期单机操作系统也是分层架构，后来才演变成 linux windows 这种内核架构，云操作系统从容器诞生之日起分层架构被击穿，未来也会朝着高内聚的"云内核"架构迁移

![image-20221017222736688](http://sm.nsddd.top/smimage-20221017222736688.png)

+ 从现在开始，把你数据中心所有机器想象成一台"抽象"的超级计算机，sealos 就是用来管理这台超级计算机的操作系统，kubernetes 就是
+ .这个操作系统的内核！
+ 云计算从此刻起再无 IaaS PaaS SaaS 之分，只有云操作系统驱动(CSI CNI CRI 实现) 云操作系统内核(kubernetes) 和 分布式应用组成

::: 

> 在这里，我将会从docker到k8s全部遍历一遍
>
> + `Docker` 的一些常用方法，当然我们的重点会在 Kubernetes 上面
> + 会用 `kubeadm` 来搭建一套 `Kubernetes` 的集群
> + 理解 `Kubernetes` 集群的运行原理
> + 常用的一些控制器使用方法
> + 还有 `Kubernetes` 的一些调度策略
> + `Kubernetes`的运维
> + 包管理工具 `Helm` 的使用
> + 最后我们会实现基于 Kubernetes 的 CI/CD



## k8s架构

容器编排系统需要满足的条件：

+ 服务注册，服务发现
+ 负载均衡
+ 配置、存储管理
+ 健康检查
+ 自动扩缩容
+ 零宕机



### 工作方式

Kubernetes采用主从分布式架构，包括Master（主节点）、Worker（从节点或工作节点），以及客户端命令行工具kubectl和其它附加项。





### 组织架构

> 我觉得尚硅谷的例子可以让我们很好的理解：

![image-20221018110649854](http://sm.nsddd.top/smimage-20221018110649854.png)



::: warning Kubernetes Control Plane
Kubernetes control Plane 负责维护集群中任何对象的 Desire State。它还管理工作节点和 Pod。它由 Kube-api-server 等五个组件组成，即 `Kube-scheduler`、`Kube-controller-manager` 和 `cloud-controller-manager`。运行这些组件的节点称为“主节点”。这些组件可以在单个节点或多个节点上运行，但建议在生产中在多个节点上运行以提供高可用性和容错性。每个控制平面的组件都有自己的职责，但是它们一起对集群做出全局决策，检测和响应由用户或任何集成的第三方应用程序生成的集群事件。

![image-20221126204020843](http://sm.nsddd.top/smimage-20221126204020843.png)

让我们了解 Kubernetes Control Plane的不同组件。Kubernetes Control Plane有以下五个组件：

+ Kube-api-server
+ Kube-scheduler
+ Kube-controller-manager
+ etcd
+ cloud-controller-manager

**Kube-API-server：**

Kube-api-server 是控制平面的主要组件，因为所有流量都通过 api-server，如果控制平面的其他组件必须与 ‘etcd’ 数据存储通信，则它们也连接到 api-server，因为只有 Kube-api-server可以与“etcd”通信。它为 REST 操作提供服务，并为 Kubernetes Control Plane提供前端，该控制平面公开 Kubernetes API，其他组件可以通过该 API 与集群通信。有多个 api-server 可以水平部署以使用负载均衡器来平衡流量。

**Kube-scheduler：**

Kube-scheduler 负责将新创建的 Pod 调度到最佳可用节点以在集群中运行。但是，可以在部署 Pod 或部署 Pod 之前，通过在 YAML 文件中指定关联性、反规范或约束，在特定节点、特定区域或根据节点标签等安排 Pod 或一组 Pod。部署。如果没有满足指定要求的可用节点，则不会部署 Pod 并保持未调度状态，直到 Kube-scheduler 找不到可行的节点。可行节点是满足 Pod 调度所有要求的节点。

Kube-scheduler 使用 2 步过程为集群中的 pod 选择节点、过滤和评分。在过滤过程中，Kube-scheduler 通过运行检查来找到一个可行的节点，比如节点是否有足够的可用资源来为这个 pod 提到。过滤掉所有可行节点后，它会根据活动得分规则为每个可行节点分配一个分数，并在得分最高的节点上运行 Pod。如果多个节点具有相同的分数，则它随机选择一个。

**Kube-controller-manager：**

Kube-controller-manager 负责运行控制器进程。它实际上由四个进程组成，并作为一个进程运行以降低复杂性。它确保当前状态与期望状态匹配，如果当前状态与期望状态不匹配，则对集群进行适当的更改以达到期望状态。

它包括节点控制器、复制控制器、端点控制器以及服务帐户和令牌控制器。

+ **节点控制器：** – 它管理节点并密切关注集群中的可用节点，并在任何节点出现故障时做出响应。
+ **复制控制器：** – 它确保为集群中的每个复制控制器对象运行正确数量的 Pod。
+ **Endpoints 控制器：** – 它创建 Endpoints 对象，例如，为了向外部公开 pod，我们需要将其加入服务。
+ **服务帐户和令牌控制器：** – 负责创建默认帐户和 API 访问令牌。例如，每当我们创建一个新命名空间时，我们需要一个服务帐户和访问令牌来访问它，因此这些控制器负责为新命名空间创建默认帐户和 API 访问令牌。

**etcd**

etcd 是 Kubernetes 的默认数据存储，用于存储所有集群数据。它是一个一致的、分布式的、高度可用的键值存储。etcd 只能通过 Kube-api-server 访问。如果其他控制平面的组件必须访问 etcd，则必须通过 kube-api-server。etcd 不是 Kubernetes 的一部分。它是由云原生计算基金会支持的完全不同的开源产品。我们需要为 etcd 设置一个适当的备份计划，这样如果集群出现问题，我们可以恢复备份并快速恢复业务。

**cloud-controller-manager**

cloud-controller-manager 允许我们将本地 Kubernetes 集群连接到云托管的 Kubernetes 集群。它是一个单独的组件，仅与云平台交互。云控制器管理器的控制器取决于我们运行工作负载的云提供商。如果我们有本地 Kubernetes 集群，或者我们为了学习目的在自己的 PC 上安装了 Kubernetes，则它不可用。cloud-controller-manager 还在单个进程中包含三个控制器，它们是节点控制器、路由控制器和服务控制器。

+ **节点控制器：** – 它不断检查托管在云提供商中的节点的状态。例如，如果任何节点没有响应，则它会检查该节点是否已在云中删除。
+ **路由控制器：** – 它在底层云基础设施中控制和设置路由。
+ **服务控制器：** – 创建、更新和删除云提供商负载均衡器。

:::




## 集群架构与组件

![img](http://sm.nsddd.top/sm1363565-20200523175956216-940931564.png)

### Master节点

Master是集群的网关和中枢枢纽，主要作用：暴露API接口，跟踪其他服务器的健康状态、以最优方式调度负载，以及编排其他组件之间的通信。单个的Master节点可以完成所有的功能，但是考虑单点故障的痛点，生产环境中通常要部署多个Master节点，组成Cluster。

| master                 | 概述                                                         |
| ---------------------- | ------------------------------------------------------------ |
| **APIServer**          | Kubernetes API，集群的统一入口，各组件协调者，以RESTful API提供接口服务，所有对象资源的增删改查和监听操作都交给APIServer处理后再提交给Etcd存储。 |
| **Scheduler**          | 根据调度算法为新创建的Pod选择一个Node节点，可以任意部署,可以部署在同一个节点上,也可以部署在不同的节点上。 |
| **Controller-Manager** | 处理集群中常规后台任务，一个资源对应一个控制器，而ControllerManager就是负责管理这些控制器的。 |

### Work Node节点

是Kubernetes的工作节点，负责接收来自Master的工作指令，并根据指令相应地创建和销毁Pod对象，以及调整网络规则进行合理路由和流量转发。生产环境中，Node节点可以有N个。

| Node                        | 概述                                                         |
| --------------------------- | ------------------------------------------------------------ |
| **kubelet**                 | kubelet是Master在Node节点上的Agent，管理本机运行容器的生命周期，比如创建容器、Pod挂载数据卷、下载secret、获取容器和节点状态等工作。kubelet将每个Pod转换成一组容器。 |
| **Pod（Docker or rocket）** | 容器引擎，运行容器。                                         |
| **kube-proxy**              | 在Node节点上实现Pod网络代理，维护网络规则和四层负载均衡工作。 |

### **etcd**数据存储

分布式键值存储系统。用于保存集群状态数据，比如Pod、Service、网络等对象信息。

### 核心附件

K8S集群还依赖一组附件组件，通常是由第三方提供的特定应用程序

| 核心插件           | 概述                                                         |
| ------------------ | ------------------------------------------------------------ |
| KubeDNS            | 在K8S集群中调度并运行提供DNS服务的Pod，同一集群内的其他Pod可以使用该DNS服务来解决主机名。K8S自1.11版本开始默认使用CoreDNS项目来为集群提供服务注册和服务发现的动态名称解析服务。 |
| Dashboard          | K8S集群的全部功能都要基于Web的UI，来管理集群中的应用和集群自身。 |
| Heapster           | 容器和节点的性能监控与分析系统，它收集并解析多种指标数据，如资源利用率、生命周期时间，在最新的版本当中，其主要功能逐渐由Prometheus结合其他的组件进行代替。从 v1.8 开始，资源使用情况的监控可以通过 Metrics API的形式获取，具体的组件为Metrics Server，用来替换之前的heapster，该组件1.11开始逐渐被废弃。 |
| Metric-Server      | Metrics-Server是集群核心监控数据的聚合器，从 Kubernetes1.8 开始，它作为一个 Deployment对象默认部署在由kube-up.sh脚本创建的集群中，如果是其他部署方式需要单独安装。 |
| Ingress Controller | Ingress是在应用层实现的HTTP(S)的负载均衡。不过，Ingress资源自身并不能进行流量的穿透，它仅仅是一组路由规则的集合，这些规则需要通过Ingress控制器（Ingress Controller）发挥作用。目前该功能项目大概有：Nginx-ingress、Traefik、Envoy和HAproxy等。 |

### 网络插件

| 网络查件                                                     | 概述                                  |
| ------------------------------------------------------------ | ------------------------------------- |
| Container Network Interface （CNI）                          | 容器网络接口                          |
| flunnal                                                      | 实现网络配置，overlay network叠加网络 |
| calico                                                       | 网络配置，网络策略；BGp协议，隧道协议 |
| canal（calico + flunnal）                                    | 供网络策略，配合flannel使用。         |
| ![img](http://sm.nsddd.top/sm1363565-20200523180136695-2145890184.png) |                                       |

## Kubernetes基本概念

| 基本概念                    |                                                              |
| --------------------------- | ------------------------------------------------------------ |
| **Label资源标签**           | 标签（key/value），附加到某个资源上，用于关联对象、查询和筛选； |
| **Labe Selector标签选择器** | 根据Label进行过滤符合条件的资源对象的一种机制                |
| **Pod资源对象**             | Pod资源对象是一种集合了一个或多个应用容器、存储资源、专用ip、以及支撑运行的其他选项的逻辑组件 |
| **Pod控制器（Controller）** | 管理Pod生命周期的资源抽象，并且它是一类对象，并非单个的资源对象，其中常见的包括：ReplicaSet、Deployment、StatefulSet、DaemonSet、Job&Cronjob等等 |
| **Service服务资源**         | Service是建立在一组Pod对象之上的资源对象，通常用作防止Pod失联、定义一组Pod的访问策略，代理Pod |
| **Ingress**                 | 如果需要将某些Pod对象提供给外部用户访问，则需要给这些Pod对象打开一个端口进行引入外部流量，除了Service以外，Ingress也是实现提供外部访问的一种方式。 |
| **Volume存储卷**            | 保证数据的持久化存储                                         |
| **Name&&Namespace**         | Name是K8S集群中资源对象的标识符，通常作用于Namespace（名称空间），因此名称空间是名称的额外的限定机制。在同一个名称空间中，同一类型资源对象的名称必须具有唯一性。 |
| Annotation注解              | 另一种附加在对象上的一种键值类型的数据；方便工具或用户阅读及查找。 |

### Label 资源标签

资源标签具体化的就是一个键值型（key/values)数据；使用标签是为了对指定对象进行辨识，比如Pod对象。标签可以在对象创建时进行附加，也可以创建后进行添加或修改。值得注意的是**一个对象可以有多个标签，一个标签页可以附加到多个对象**。

![img](http://sm.nsddd.top/sm1363565-20200523180226573-1554114165.png)

### Labe Selector标签选择器

有标签，当然就有标签选择器；例如将含有标签`role: backend`的所有Pod对象挑选出来归并为一组。通常在使用过程中，会通过标签对资源对象进行分类，然后再通过标签选择器进行筛选，最常见的应用就是为一组同样标签的Pod资源对象创建某个Service的端点。

![img](http://sm.nsddd.top/sm1363565-20200523180332039-330736525.png)

### Pod资源对象

Pod是kubernetes的最小调度单元；是一组容器的集合

> Pod可以封装一个活多个容器！同一个Pod中共享网络名称空间和存储资源，而容器之间可以通过本地回环接口：lo 直接通信，但是彼此之间又在Mount、User和Pid等名称空间上保持了隔离。

Pod其实就是一个应用程序运行的单一实例，它通常由共享资源且关系紧密的一个或2多个应用容器组成。

![img](http://sm.nsddd.top/sm1363565-20200523180259373-1808638376.png)

我们将每一个Pod对象类比为一个物理主机，那么运行在同一个Pod对象中的多个进程，也就类似于物理主机上的独立进程，而不同的是Pod对象中的各个进程都运行在彼此隔离的容器当中，而各个容器之间共享两种关键性资源;

网络&&存储卷。

+ 网络：每一个Pod对象都会分配到一个Pod IP地址，同一个Pod内部的所有容器共享Pod对象的Network和UTS名称空间，其中包括主机名、IP地址和端口等。因此，这些容器可以通过本地的回环接口lo进行通信，而在Pod之外的其他组件的通信，则需要使用Service资源对象的Cluster IP+端口完成。
+ 存储卷：用户可以给Pod对象配置一组存储卷资源，这些资源可以共享给同一个Pod中的所有容器使用，从而完成容器间的数据共享。存储卷还可以确保在容器终止后被重启，或者是被删除后也能确保数据的持久化存储。

一个Pod代表着一个应用程序的实例，现在我们需要去扩展这个应用程序；那么就意味着需要创建多个Pod实例，每个实例都代表着应用程序的一个运行副本。

而管理这些副本化的Pod对象的工具，都是由一组称为Controller的对象实现；例如Deployment控制器对象。

当创建Pod时，我们还可以使用Pod Preset对象为Pod注入特定的信息，比如Configmap、Secret、存储卷、卷挂载、环境变量等。有了Pod Preset对象，Pod模板的创建就不需要为每个模板显示提供所有信息。

基于预定的期望状态和每个Node节点的资源可用性；Master会把Pod对象调度至选定的工作节点上运行，工作节点从指向的镜像仓库进行下载镜像，并在本地的容器运行时环境中启动容器。Master会将整个集群的状态保存在etcd中，并通过API Server共享给集群的各个组件和客户端

### Pod控制器（Controller）

在介绍Pod时我们提到，Pod是K8S的最小调度单位；但是Kubernetes并不会直接地部署和管理Pod对象，而是要借助于另外一个抽象资源——Controller进行管理。

常见的Pod控制器：

| Pod Controller             |                                                              |
| -------------------------- | ------------------------------------------------------------ |
| **Replication Controller** | 使用副本控制器，早起仅支持此Pod控制器；完成Pod增减、总数控制、滚动更新、回滚等操作，已停止使用 |
| **ReplicaSet Controller**  | 版本更新后使用副本集控制器，并对使用方法进行声明；是Replication Controller的升级版 |
| **Deployment**             | 用于无状态应用部署，例如nginx等；后续我们会提到HPA Controller（Horizontal Pod Autosaler）：用于水平Pod自动伸缩控制器，对rs&deployment进行控制 |
| **StatefulSet**            | 用于有状态应用部署，例如mysql，zookeeper等                   |
| **DaemonSet**              | 确保所有Node运行同一个Pod，例如网络查件flannel，zabbix_agent等 |
| Job                        | 一次性任务                                                   |
| Cronjob                    | 定时任务                                                     |

控制器是更高级层次对象，用于部署和管理Pod。

以Deployment为例，它负责确保定义的Pod对象的副本数量符合预期的设置，这样用户只需要声明应用的期望状态，控制器就会自动地对其进行管理。

![img](http://sm.nsddd.top/sm1363565-20200523180401866-1621029241.png)

用户通过手工创建或者通过Controller直接创建的Pod对象会被调度器（Scheduler）调度到集群中的某个工作节点上运行，等到容器应用进程运行结束之后正常终止，随后就会被删除。

> 当节点的资源耗尽或者故障，也会导致Pod对象的回收。

在K8S的集群设计中，Pod是一个有生命周期的对象。那么使用了控制器实现对一次性的Pod对象进行管理操作。

> 例如，要确保部署的应用程序的Pod副本数达到用户预期的数目，以及基于Pod模板来重建Pod对象等，从而实现Pod对象的扩容、缩容、滚动更新和自愈能力。
>
> 例如，在某个节点故障，相关的控制器会将运行在该节点上的Pod对象重新调度到其他节点上进行重建。

控制器本身也是一种资源类型，它们都统称为Pod控制器。如下图的Deployment就是这类控制器的代表实现，是目前用于管理无状态应用的Pod控制器。

![img](http://sm.nsddd.top/sm1363565-20200523180431487-339597555.png)

Pod Controller的定义通常由期望的副本数量、Pod模板、标签选择器组成。Pod Controller会根据Labe Selector来对Pod对象的标签进行匹配筛选，所有满足选择条件的Pod都会被当前Controller进行管理并计入副本总数，确保数目能够达到预期的状态副本数。

> 在实际的应用场景中，在接收到的请求流量负载低于或接近当前已有Pod副本的承载能力时，需要我们手动修改Pod控制器中的期望副本数量以实现应用规模的扩容和缩容。而在集群中部署了HeapSet或者Prometheus的这一类资源监控组件时，用户还可以通过HPA（HorizontalPodAutoscaler）来计算出合适的Pod副本数量，并自动地修改Pod控制器中期望的副本数，从而实现应用规模的动态伸缩，提高集群资源的利用率。

K8S集群中的每个节点上都运行着`cAdvisor`，用于收集容器和节点的CPU、内存以及磁盘资源的利用率直播数据，这些统计数据由Metrics聚合之后可以通过API server访问。而`HorizontalPodAutoscaler`基于这些统计数据监控容器的健康状态并作出扩展决策。

### Service服务资源

| 主要作用or功能                 |                                                              |
| ------------------------------ | ------------------------------------------------------------ |
| 防止Pod失联                    | Service是建立在一组Pod对象之上的资源对象，在前面提过，它是通过Labe Selector选择一组Pod对象，并为这组Pod对象定义一个统一的固定访问入口（通常是一个IP地址），如果K8S存在DNS附件（如coredns）它就会在Service创建时为它自动配置一个DNS名称，用于客户端进行服务发现。 |
| 定义一组Pod的访问策略，代理Pod | 通常我们直接请求Service IP，该请求就会被负载均衡到后端的端点，即各个Pod对象，即负载均衡器；因此Service本质上是一个4层的代理服务，另外Service还可以将集群外部流量引入至集群，这就需要节点对Service的端口进行映射了。 |

Pod对象有Pod IP地址，但是该地址在对象重启或者重建之后随之改变，Pod IP地址的随机性给应用系统依赖关系维护创造了不小的麻烦。

> 例如：前端Pod应用`Nginx`无法基于固定的IP地址负载后端的Pod应用`Tomcat`。

Service资源就是在被访问的Pod对象中添加一个有着固定IP地址的中间层，客户端向该地址发起访问请求后，由相关的Service资源进行调度并代理到后端的Pod对象。

Service并不是一个具体的组件，而是一个通过规则定义出由多个Pod对象组成而成的逻辑集合，并附带着访问这组Pod对象的策略。Service对象挑选和关联Pod对象的方式和Pod控制器是一样的，都是通过标签选择器进行定义。

![img](http://sm.nsddd.top/sm1363565-20200523180459175-924096694.png)

------

Service IP是一种虚拟IP，也称为`Cluster IP`，专用于集群内通信

> 通常使用专有的地址段，如：10.96.0.0/12网络，各Service对象的IP地址在该范围内由系统动态分配。

集群内的Pod对象可直接请求这类`Cluster IP`，比如上图中来自Pod client的访问请求，可以通过Service的`Cluster IP`作为目标地址进行访问，但是在集群网络中是属于私有的网络地址，**只能在集群内部访问**。

通常我们需要的是外部的访问；将引入集群内部的常用方法是通过节点网络进行，其实现方法如下：

> + 通过工作节点的IP地址+端口（Node Port）接入请求。
> + 将该请求代理到相应的Service对象的Cluster IP的服务端口上，通俗地说：就是工作节点上的端口映射了Service的端口。
> + 由Service对象将该请求转发到后端的Pod对象的Pod IP和 应用程序的监听端口。

因此，类似于上图来自External Client的集群外部客户端，是无法直接请求该Service的Cluster IP，而是需要实现经过某一工作节点Node的IP地址，此时请求需要2次转发才能到目标Pod对象。这一类访问的缺点就是在通信效率上有一定的延时。

### Ingress

K8S将Pod对象和外部的网络环境进行了隔离，Pod和Service等对象之间的通信需要通过内部的专用地址进行

例如果需要将某些Pod对象提供给外部用户访问，则需要给这些Pod对象打开一个端口进行引入外部流量，除了Service以外，Ingress也是实现提供外部访问的一种方式。

### Volume存储卷

存储卷（Volume）是独立于容器文件系统之外的存储空间，常用于扩展容器的存储空间并为其提供持久存储能力。

> 存储卷在K8S中的分类为：
>
> 1. 临时卷
> 2. 本地卷
> 3. 网络卷

临时卷和本地卷都位于Node本地，一旦Pod被调度至其他Node节点，此类型的存储卷将无法被访问，因为临时卷和本地卷通常用于数据缓存，持久化的数据通常放置于持久卷（persistent volume）之中。

### Name和Namespace

名称空间通常用于实现租户或项目的资源隔离，从而形成逻辑分组。关于此概念可以参考Docker文档中的概念https://www.jb51.net/article/136411.htm

如图：创建的Pod和Service等资源对象都属于名称空间级别，未指定时，都属于默认的名称空间`default`

![这个图片挂了⚠️ ~](http://sm.nsddd.top/sm1363565-20200523180512841-2018842328.png)

### Annotation注解

Annotation是另一种附加在对象上的一种键值类型的数据，常用于将各种非标识型元数据（metadata）附加到对象上，但它并不能用于标识和选择对象。其作用是方便工具或用户阅读及查找。
