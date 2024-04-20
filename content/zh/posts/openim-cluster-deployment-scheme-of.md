---
title: 'OpenIM 的集群化设计 | Kubernetes 部署 | 方案讨论 | 会议总结'
ShowRssButtonInSectionTermList: true
cover.image:
date: 2023-09-17T09:51:54+08:00
draft: false
showtoc: true
tocopen: true
type: posts
author: '熊鑫伟，我'
keywords: ['OpenIM', 'Kubernetes', '集群化', '设计', '部署', '方案']
tags: ['OpenIM', 'Kubernetes', '集群化 (Clustering)', '云原生 (Cloud Native)', '微服务 (Microservices)', '部署 (Deployment)']
categories: ['开发 (Development)','项目管理 (Project Management)']
description: '本文详细总结了在 Kubernetes 环境下部署 OpenIM 集群化解决方案的过程，探讨了关键设计考虑、部署策略及其挑战，旨在为同类项目提供实践指南和设计启示。'
---

## 会议和参考链接

+ 会议参考文档：[https://nsddd.notion.site/2899028707604b8090b36677c031cdf8?pvs=4](https://nsddd.notion.site/2899028707604b8090b36677c031cdf8?pvs=4)
+ 视频回放：Bilibili: [https://www.bilibili.com/video/BV1s8411q7Um/?spm_id_from=333.999.0.0](https://www.bilibili.com/video/BV1s8411q7Um/?spm_id_from=333.999.0.0)



**评论：**

+ 那个中间件我觉得可以换成 https://kubeblocks.io 可以帮你管理多个数据库中间件
+ im 读取配置信息，读取的是 config/ 目录，代码中硬编码补充的 config.yaml，是否可以自动化来对 不同 服务的 rpc 划分，然后统一目录，默认读取的是二进制运行路径的上两层
+ openim version: https://github.com/openimsdk/open-im-server/blob/main/docs/conversions/version.md
+ 存储可以考虑使用 ：
  + https://github.com/openebs/openebs
  + https://github.com/rook/rook 



**核心目标：**

开源项目和非开源项目的最大的区别，就是一套完整的解决方案。

+ 非开源项目的 集群化部署方案的设计，比较在乎稳定，以及高效，快速，最好一键部署。
+ 开源项目的 集群化部署方案的设计，比较在乎通用性，上手的难度，后期的维护难度，基础架构的稳定性。后面的开发者或者是贡献者，使用者，可以基于此创建自己的集群化部署方案，以及解决方案，并且成为 OpenIM 的集群化部署方案的使用案例。



## OpenIM 集群化部署讨论会记录

先总结，后详细解读  ~

### 关于开源部署环境的演变与变化

+ **新部署方法**：一种集二进制和部署于一体的一键操作。
+ **Kubernetes 部署**：在 Kubernetes 环境中实现一键部署的新型方案。
+ **现存问题**：涉及日志收集、服务重启追踪等，将在后续对这些问题进行改进并寻求解决方案。

### CICD的开发与维护策略

+ **CICD 概念**：通过CICD实现Code Streaming。
+ **开发阶段**：需要编写出镜像文件。**GitHub的CSD功能**：已实现但尚待深入研究。
+ **版本标记策略**：推荐使用local branch而非直接标签。

### 关于软件开发与测试的实践经验分享

+ **本地开发**：推荐使用“auto-compile”工具快速生成稳定版本的镜像。
+ **团队协作**：介绍了各团队间如何协同进行开发、测试和发布。
+ **代码重用**：提及将库中的函数或方法封装为组件，实现跨项目调用。

### Docker Deployment与Service Configuration

+ **配置传递**：主要通过如K8S中的配置文件。
+ **部署方式**：介绍了二进制部署和可部署两种策略，并讨论了各自的优缺点。

### 关于容器化部署和代码优化的探讨

+ **容器化**：提议将多个进程合并为一个容器进行管理。
+ **部署方式兼容性**：讨论了如何实现并进行微调。
+ **技术架构和组件**：如Helm chat、OpenM等，及其在系统中的作用和重要性。

### 关于一键部署的技术问题与解决方案

+ **一键部署问题**：可能的问题有无法翻墙、无法安装等。
+ **解决方案**：1) 将现有方案通用化；2) 采用第三方服务实现一键部署。

### K8S部署与自动化的优化策略

+ **部署工具**：如使用Shell实现一键部署、K ks部署等。
+ **组件整合**：考虑如何将不同组件组合成完整解决方案，并保持不同环境中的一致性。

### 微服务架构中的最佳实践

+ **应用程序部署**：建议将应用程序划分为不同的容器，每个容器内运行一个业务进程。
+ **代码整合**：提议将相关代码整合为一个文件进行管理。

### 微服务的优化与部署策略

+ **微服务划分**：强调避免过于细致的模块分割。
+ **自动化**：部署时不增加额外维护工作量，采用自动化策略。

### 关于存储方式和编排工具的选择

+ **文件存储**：如使用NFS作为本地分布式文件存储。
+ **编排工具**：推荐使用 rook 进行对象存储编排，数据库使用专用编排器。

### NFS与Flexible File System的应用

+ **苹果手机上的MFS**：讨论了其使用情况和如何同步全局配置文件到各业务模块。
+ **PV/PVC管理数据**：示例讲解如何使用此文件系统进行数据管理。

### 二进制代码与配置文件的应用

+ **代码适配**：通过配置文件进行，涉及传递配置路径、文件映射等细节。

### 关于软件开发中的优化与改进

+ **项目脚本编写**：讨论了性能瓶颈、部署统一处理、服务发现模块的优化建议。

### 关于Web应用配置文件的编写与优化

+ **IP分配**：配置文件用于IP分配和模块间分段处理。
+ **接口应用**：如在不同环境使用不同的接口实现心跳等功能。
+ **技术架构改进**：优化轻量化、提高开发效率和维护效果等。

**结论**：本次讨论会涉及了开源部署环境的多个方面，从软件开发、部署、测试到微服务架构和存储方式等多个领域。希望通过此次讨论，可以为OpenIM的集群化部署提供有力的参考和指导。



## Kubernetes 集群设计方案

### Ingress-Controller 的选择

为了提供一个可伸缩和灵活的环境，我们打算使用以下Ingress-Controller：

+ **开发和初期阶段**： 使用`nginx-controller`。

  *理由*：简单，快速，易于配置，适合早期开发和测试。

+ **生产和扩展阶段**：考虑使用`traefik`或`istio`。

  *理由*：为了满足生产环境的复杂性和可扩展性需求。

### 基础组件层的部署

我们将使用`Helm charts`来部署以下基础组件：

+ MySQL
+ Redis
+ MongoDB
+ Kafka
+ Loki
+ Prometheus
+ Grafana

*理由*：`Helm`能够简化 Kubernetes 应用程序的部署、升级和管理，使得基础组件的部署更加简洁。

### 应用层的设计

对于`openim-server`和`openim-chat`，考虑以下策略：

+ 为`openim-server`和`openim-chat`的每一个模块都建立单独的`Helm chart`。

  *理由*：这样可以方便地收集日志、监控以及重启的状态管理。

### openim-server 和 openim-chat 的 K8s 适配

+ 现有的通过 zookeep 服务发现将被替换，改为通过K8s的`servicename`域名通信。

  *理由*：在Kubernetes环境中，使用servicename进行服务发现更加直观，且易于管理和扩展。



##  OpenIM 集群通用设计思路



### 整体思想

+ **部署方式**：
  + 二进制（已实现）
  + docker-compose（已实现）
  + k8s部署（目标为一键部署，配合 sealos）
  + [openim-docker GitHub地址](https://github.com/openimsdk/openim-docker)
+ **代码与部署适配**：同一份业务代码应适配三种部署方式，差异仅在部署脚本。具体可通过`install.sh`的传参和环境变量进行扩展性设计。
+ **服务进程独立化**：考虑将`openim-server`和`openim-chat`容器内的多个进程进行独立成容器，目的为简化日志收集、容易追踪服务重启与panic、便捷的监控以及精细化的扩容。这也更契合微服务的思想。
+ **CICD流程**：开发阶段和正式发布阶段的镜像`tag`需要有所区别。开发阶段编译的镜像`tag`为分支名；正式发布的`tag`为`release`加版本号；提测阶段`tag`名称为`rc0`,`rc1`,`rc2`...。这样便于开发阶段进行速度更新。
+ **服务配置策略**：全部服务的配置信息都应通过`yaml`配置文件传递，具体的启动命令为 `openim -c /data/openim/config.yaml`。默认配置为从`/data/服务名称/config.yaml`读配置文件。配置文件应涵盖全部运行所需的配置信息。
+ **部署脚本细节**：
  + `docker-compose`：继续使用`shell+compose.yaml`策略，为每个服务抽取一个`config.yaml`配置文件，并映射进容器的`/data/openim/`目录。
  + `k8s`：采用`shell+helm chart`策略。基础架构组件在我们自己的helm repo中维护一个开源稳定版本和默认配置`value.yaml`。所有的基础服务配置应维护在一个全局配置文件`openim.yaml`内，用于覆盖默认`value.yaml`。

### 各helm chart编写

| **分类**                | **包含**                                                     | **说明**                                                     | **备注**                                        |
| ----------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ----------------------------------------------- |
| ingress-controller      | nginx-ingress                                                | 目前主流的ingress-controller有三个：istio，traefik，nginx。推荐后期过度到traefik。 | 配置http和ws走相同端口。                        |
| 业务服务模块            | openim-api, openimmsg-gateway, openim-push, openim-msgtransfer, openim-rpc-*, 前端模块 | 建议对轻量级负责数据库存储的rpc服务进行合并到openim-api。    | 服务分太细会增加维护性。                        |
| 基础架构模块            | mysql, redis, mongodb, kafka, loki, Prometheus, grafana, zookeeper | 维护一个稳定的开源helm chart和默认value。                    | 推送到我们自己的helm repo，便于管理和用户安装。 |
| 全局配置文件openim.yaml | 覆盖所有业务模块helm的value.yaml                             | 抽象全局相同的配置信息。                                     | 如：基础架构账户，url信息，pvc路径映射信息等。  |
| shell脚本               | 对服务选择性进行Installation，restart，delete。              | 脚本需捕获helm安装成功或失败的返回值。                       | 完成基础的模板化和自动化功能。                  |

**Helm chart目录结构**：[查看链接](https://wdcdn.qpic.cn/MTY4ODg1NDc5OTQ3MTMwMw_344519_vqHmjF7xlPMUz1ht_1694910900?sign=1694915159-592367056-0-b8360271e7d98a1b531eb3a5713ceb85)

### 应用配置文件适配

+ 二进制部署：通过`-c /data/openim/config.yaml`命令行参数传递。
+ docker-compose：映射配置文件进容器的`/data/服务名称/`目录。
+ k8s：创建configmap，并在deployment中映射至容器内。

### 应用服务发现与服务注册的适配

由于k8s自带服务发现和服务注册机制，考虑进行适配以简化部署。具体做法是在`discoveryregistry.SvcDiscoveryRegistry`上再封装一层，对于docker-compose和二进制部署，维持原来流程。若为k8s部署，则使用服务名的内部域名进行通信。

### 修改点

+ **CICD**：使用github actions实现。构建流程要使得在`dev`,`test`,`release`三种环境中生成对应的镜像tag。
+ **部署功能**：维护三套部署脚本和对应的yaml配置文件。
+ **开发建议**：微服务划分不宜过多。建议选择轻量级的云原生模块，并对公共模块进行独立维护。



## 设计步骤

**OpenIM** 是一个开源的即时通讯解决方案。为了保障其在大规模应用场景下的高可用性和高性能，集群化的部署设计，侧重于为OpenIM提供一个专业、完整的集群化设计指南，涵盖从基础架构、持续集成/部署到微服务化优化的所有关键步骤。

### 基础架构设计

#### 网络设计

+ **子网规划**：确保每个可用区(AZ)有其独立的子网，并保证其之间的隔离性。
+ **Load Balancer**：使用云提供商或开源负载均衡器（如 Nginx、HAProxy）以保障高可用性和流量分发。

#### 存储设计

+ **持久化存储**：利用云原生的持久化解决方案，如 AWS EBS、GCE Persistent Disk 或开源的 Rook。
+ **日志与监控**：集成ELK (Elasticsearch, Logstash, Kibana) 或 EFK (Elasticsearch, Fluentd, Kibana) 堆栈，确保日志的实时收集、分析和展示。



### CI/CD & GitOps

#### 持续集成

+ **编译与测试**：集成 Jenkins、GitLab CI 或 GitHub Actions，确保每次代码提交后进行自动化的单元测试和构建。

#### 持续部署

+ **部署流程**：确保每次成功的构建可以自动推送到测试环境，并有流程支持自动或半自动推送到生产环境。
+ **配置管理**：利用 Helm 或 Kustomize，实现应用配置的版本化管理与自动部署。

#### GitOps

+ 采用 ArgoCD 或 Flux，实现声明式的应用部署。确保所有集群的变更都可以通过 Git 追踪。



### 容器化与服务编排

#### 容器设计

+ 使用 Docker 作为容器解决方案，确保服务的隔离性和一致性。
+ 镜像存储在私有或公开的容器仓库中，如 Docker Hub、Quay.io 或云提供商的容器仓库。

#### Kubernetes 作为服务编排工具

+ **多集群管理**：考虑使用 Rancher 或 Kubefed，实现跨多个集群的统一管理。
+ **网络策略**：利用 Calico 或 Cilium，为 Pod 间通信实现网络策略和安全。



###  微服务化优化

#### 服务划分

+ **功能分离**：确保每个微服务只做一件事，并做好它。
+ **通信**：采用 gRPC 或 RESTful API 作为微服务间的通信方式。

#### 服务发现与负载均衡

+ 利用 Istio 或 Linkerd，为微服务提供服务网格功能，实现服务发现、负载均衡、灰度发布等高级功能。

#### 限流与熔断

+ 使用 Hystrix 或 Sentinel，为微服务提供限流、熔断和降级策略。



###  监控与告警

#### 监控

+ 使用 Prometheus 和 Grafana，提供实时的监控数据展示。

#### 日志

+ 利用 Loki 或 Fluentd，为集群提供日志聚合功能。

#### 告警

+ 结合 Alertmanager 或 ElastAlert，确保在关键问题发生时能及时通知相关团队。



### 安全

#### 网络安全

+ **Pod 网络策略**：利用 NetworkPolicies 限制 Pod 之间的不必要通信。
+ **入口/出口流量**：使用 Istio 的 egress 和 ingress gateway 控制集群的流入和流出流量。

#### IAM

+ 使用 OpenID Connect 或 Dex，为 Kubernetes 提供统一身份验证。