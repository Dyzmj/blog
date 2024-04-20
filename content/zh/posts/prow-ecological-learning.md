---
title: 'Prow 是什么？kubernetes 为什么需要它'
description: 'Prow 是一个基于 Kubernetes 的 CI/CD 系统，它可以在 Kubernetes 集群中运行，用于管理和执行 CI/CD 流程。'
ShowRssButtonInSectionTermList: true
cover.image:
date : 2023-09-16T16:46:15+08:00
draft : false
showtoc: true
tocopen: true
author: ["熊鑫伟", "Me"]
keywords: []
tags:
  - blog
categories:
  - Development
  - Blog
---

## why?

**故事是从这个 proposal 开始idea~**

🤖 [OpenIM cicd robot machine proposal](https://github.com/OpenIMSDK/Open-IM-Server/issues/398)

Prow是基于Kubernetes的CI/CD系统。作业可以由各种类型的事件触发，并将其状态报告给许多不同的服务。除了作业执行，Prow还以策略执行、通过 `/foo` 风格命令的聊天操作和自动PR合并的形式提供GitHub自动化。

有关 Golang 文档，请参阅 [GoDoc](https://pkg.go.dev/k8s.io/test-infra/prow)。请注意，这些库仅供prow使用，我们不会尝试保留向后兼容性。

**Kubernetes 专门为 Prow 提供了网页命令查询：**

+ [https://prow.k8s.io/command-help](https://prow.k8s.io/command-help)

关于Prow如何运行作业的简要概述，请参阅 [Prow作业的生命周期](https://docs.prow.k8s.io/docs/life-of-a-prow-job/)。

要查看Prow的常用用法和交互流，请参见拉取请求交互序列图。



### hello world

最简单的一个上手案例莫过于 [pull request](https://help.github.com/articles/about-pull-requests/) 。

提出一个拉取请求（以下简称PR）。在PR正文中，可以随意添加区域标签（如果合适），例如 `/area <AREA>` 。标签列表[在这里](https://github.com/kubernetes/test-infra/labels)。也可以随意推荐一位评论者 `/assign @theirname` 。

一旦你的审阅者满意，他们会说 `/lgtm` ，这将应用 `lgtm` 标签，如果他们是OWNER，将应用 `approved` 标签。 `approved` 标签也将自动应用于所有者打开的PR。如果你和你的审阅者都不是OWNER，请 `/assign` 某个所有者。如果你的PR有 `lgtm` 和 `approved` 标签，没有任何 `do-not-merge/*` 标签，并且所有测试均通过，则PR将自动合并。



### 查看测试结果

+ Kubernetes TestGrid 显示历史测试结果
  + 在 [testgrid/config.yaml](https://github.com/k8s-ci-robot/test-infra/blob/master/testgrid/config.yaml) 配置自己的 testgrid 仪表盘
  + [Gubernator](https://gubernator.k8s.io/) 格式化每次运行的输出
+ [PR Dashboard](https://gubernator.k8s.io/pr) 查找需要注意的 PR
+ Prow 安排测试并更新问题
  + Prow 响应 GitHub 事件、定时器和在 GitHub 评论中给出的[手动命令](https://go.k8s.io/bot-commands)。
  + [prow dashboard](https://prow.k8s.io/) 显示当前正在测试什么
  + 在 [config/jobs](https://github.com/k8s-ci-robot/test-infra/blob/master/config/jobs) 配置 prow 运行新测试
+ Triage Dashboard 汇总故障
  + 将故障集群在一起
  + 搜索跨作业的测试失败
  + 在特定的测试和/或作业的 regex 中过滤失败
+ Velodrome 指标跟踪作业和测试健康状况。
  + [Kettle](https://github.com/k8s-ci-robot/test-infra/blob/master/kettle) 进行收集，[metrics](https://github.com/k8s-ci-robot/test-infra/blob/master/metrics) 进行报告，[velodrome](https://github.com/k8s-ci-robot/test-infra/blob/master/velodrome) 是前端。



## 功能和特性

**prow 的功能很强大，甚至是比 actions 更加出众。可以测试、批处理、工件发布的作业执行。**

+ GitHub事件用于触发post-PR-merge（postsubmit）作业和on-PR-update（presubmit）作业。
+ 支持多个执行平台和源代码审查站点。

**可插拔的GitHub bot自动化，实现 `/foo` 风格的命令并强制执行配置的策略/流程。**

::: details 什么是 foo 风格 
`/foo` 风格通常指的是在聊天应用程序（如Slack）中使用的命令格式。这种格式的命令以斜杠 `/` 开头，后跟一个关键字或短语，例如 `/help` 或 `/status`。GitHub bot自动化可以使用此格式来实现特定功能，例如在GitHub上自动创建问题或拉取请求，并强制执行特定的工作流程或策略。

:::



**其他的功能：**

+ GitHub将自动化与批量测试逻辑合并。
+ 用于查看作业、合并队列状态、动态生成的帮助信息等的前端。
+ 基于配置的源代码管理的自动部署。
+ 在源代码控制中配置的自动 `GitHub org/repo` 管理。
+ 专为具有数十个存储库的多组织规模而设计。(The Kubernetes Prow实例仅使用1个GitHub bot令牌！）
+ 高可用性是在Kubernetes上运行的好处。（复制、负载平衡、滚动更新...）
+ JSON结构化日志。
+ 支持 普罗米修斯指标。



## Documentation

任何一个大型的工程第一个考虑的应该是稳定性，prow的稳定性很大程度上依赖于单元测试和集成测试。

+ 单元测试与prow源代码位于同一位置
+ [Integration tests](https://docs.prow.k8s.io/docs/test/integration/) utilizes [kind](https://kind.sigs.k8s.io/) with hermetic integration tests. See [instructions for adding new integration tests](https://docs.prow.k8s.io/docs/test/integration/#adding-new-integration-tests) for more details



**Getting started：我们分为三个大的板块**

+ 使用自己的 Prow 部署
+ 为 Prow 开发
+ As a job author: [ProwJobs](https://docs.prow.k8s.io/docs/jobs/)



## 使用自己的Prow部署

这里我们应该学会的是如何将你自己的 Prow 实例部署到 Kubernetes 的集群。

Prow可以在任何`Kubernetes`集群中运行。下面的指南专注于Google Kubernetes Engine，但应该适用于任何Kubernetes发行版，**无需/只需** 很少的更改。

Prow 是使用 webhook 做的，webhook 相对来说比较复杂一些。

就是当你创建 robot 的时候，先创建 webhook，然后写入 webhook 链接。

k8s有自己的服务，prew 需要单独搭建出来。

> webhook的地址是你写的服务的公网地址。相当于github要调用webhook

其实你写个webhook然后把webhook搭建到cloud上，就串起来了。然后你自动发布就直接改镜像完事。主要就是把webhook写一下，解析指令就行了。

如果你没有自己的服务器，也可以借助 actions 自己跑，参考：[https://github.com/labring/sealos/blob/main/.github/workflows/bot.yml](https://github.com/labring/sealos/blob/main/.github/workflows/bot.yml)



### GitHub APP

首先，你需要 [创建一个GitHub应用程序](https://docs.github.com/en/apps/creating-github-apps/setting-up-a-github-app/creating-a-github-app)。GitHub本身也记录了这一点。 最初，为Webhook设置一个虚拟URL就足够了。所需的确切权限集因你使用的功能而异。下面是所需的最低权限集。

> ⚠️ 一个用户或者组织最多只能有 100 个 robot

**存储库权限**：

+ Actions：只读（仅在使用合并自动化 `tide` 时需要）
+ Administration：只读（获取团队和协作者时必需）
+ Checks：只读（仅在使用合并自动化 `tide` 时需要）
+ Contents：读取（使用合并自动化时需要读取和写入 `tide` ）
+ Issues: Read & write 
+ Metadata: Read-Only
+ Pull Requests: Read & write
+ projects：使用 `projects` 插件时为Admin，否则为none
+ Commit statuses: Read & write



**组织权限：**

+ Members：只读（使用 `peribolos` 时读写）
+ projects：使用 `projects` 插件时为Admin，否则为none



在 `Subscribe to events` 中选择所有事件。

保存应用程序后，单击底部的“生成私钥”，并将私钥与页面顶部的 `App ID` 一起保存。

sealos 也集成了自己的 robot，可以作为发行版或者是解决平常的 PR

+ **[sealos-release-rebot](https://github.com/sealos-release-rebot)**
+ **[k8s-release-robot](https://github.com/k8s-release-robot)**

关于 `sealos` 的 bot 仓库地址，藏在了[这里](https://github.com/labring/gh-rebot) 



## 如何做一个 github-bot

GitHub机器人是一种自动化工具，它可以在服务端上启动一个基于 [Koa.js](https://github.com/koajs/koa/) 的HTTP服务器，并建立一些项目规范，例如规定issue格式、pull request格式、配置指定标签的所有者、统一git commit log格式等。通过使用 [GitHub Webhooks](https://github.com/labring/sealos/blob/main/.github/workflows/bot.yml) 和 [GitHub API](https://docs.github.com/en/rest)，机器人可以自动处理一些事情，例如自动回复issue、自动合并pull request等。通常情况下，机器人是一个单独的账号，例如 [@kubbot](https://github.com/kubbot)。使用GitHub机器人可以实现快速响应、自动化和解放人力的效果，从而提高项目的效率和质量。



### actions 关闭和操作 issue 

其实 robot 最基本的权限就是对 issue 和 PR 的权限。

```bash
name: Invite users to join our group
on:
  issue_comment:
    types:
      - created
jobs:
  issue_comment:
    name: Invite users to join our group
    if: ${{ github.event.comment.body == '/invite' }}
    runs-on: ubuntu-latest
    permissions:
      issues: write
    steps:

      - name: Invite user to join our group
        uses: peter-evans/create-or-update-comment@v1
        with:
          issue-number: ${{ github.event.issue.number }}
          body: |
			#......

      - name: Close Issue
        uses: peter-evans/close-issue@v3
        with:
          issue-number: ${{ github.event.issue.number }}
          comment: auto-closing issue, if you still need help please reopen the issue or ask for help in the community above
          labels: |
            triage/accepted
```



**github 地址：**

+ [https://github.com/peter-evans/close-issue](https://github.com/peter-evans/close-issue)

最后还贴心的加上 labels



### 再来一个：Issues Translate Chinese Action

将包含中文 issue 实时翻译成英文 issue 的 action。

```yaml
name: 'issue translator'
on:
  issue_comment:
    types: [created]
  issues:
    types: [opened]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: usthe/issues-translate-action@v2.7
        with:
          # it is not necessary to decide whether you need to modify the issue header content
          IS_MODIFY_TITLE: true
          BOT_GITHUB_TOKEN: ${{ secrets.BOT_GITHUB_TOKEN }}
          # Required, input your bot github token这里有一点很神奇，就是我们不需要去指定 GitHub 中 BOT 的用户 @kubbot ，GitHub在环境密钥中就可以知道、解析出来。
```

用了上面的模板后，GitHub就能自动的去分析 issue 并且翻译。

**还有一个是用的 chatgpt 翻译的：**

+ [https://github.com/marketplace/actions/gpt-translate](https://github.com/marketplace/actions/gpt-translate)

在问题或拉取请求中创建带有 `/gpt-translate` 或 `/gt` 的评论。

[On issue]转换后的文件将作为拉取请求创建。

[On pull request]转换后的文件将通过新提交添加到pull request中。

换句话说，如果你继续评论一个问题，新的PR将不断被创建。如果你一直在PR上评论，新的提交将不断地被添加到该PR中。

```bash
/gpt-translate README.md README_zh-CN.md traditional-chinese
```

actions 文件：

[@kubbot](https://github.com/kubbot)

```yaml
# .github/workflows/gpt-translate.yml
name: GPT Translate

on:
  issue_comment:
    types: [ created ]

jobs:
  gpt_translate:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Run GPT Translate
        if: |
          contains(github.event.comment.body, '/gpt-translate') || 
          contains(github.event.comment.body, '/gt')
        uses: 3ru/gpt-translate@v1.0
        with:
          apikey: ${{ secrets.OPENAI_API_KEY }}
          token: "${{ secrets.BOT_GITHUB_TOKEN }}"
```





### 如何使用指定的GitHub robot 代替 GitHub actions 

每一次看到 GitHub actions，感觉看着不顺眼，还不如搞一个自己的 robot。

但是一不小心搞了两个 😒

+ https://github.com/kubbot: I am a member bot of [@OpenIMSDK](https://github.com/OpenIMSDK) and the older brother of [@openimbot](https://github.com/openimbot)
+ https://github.com/openimbot: I am a member bot of [@OpenIMSDK](https://github.com/OpenIMSDK) openim and a sister of [@kubbot](https://github.com/kubbot)



### Lighthouse

+ https://github.com/jenkins-x/lighthouse

Lighthouse是一个轻量级的基于 ChatOps 的 webhook 处理程序，可以基于来自多个git提供商（如GitHub，GitHub Enterprise，BitBucket Server和GitLab）的webhook触发Jenkins X Pipelines，Tekton Pipelines或Jenkins Jobs。

 Lighthouse 最开始的时候也是基于 prow 的，并且是从他们的基础代码的副本开始。

目前，Lighthouse支持标准的Prow插件，并处理将webhook推送到分支，然后在你选择的代理上触发管道执行。

Lighthouse使用与Prow相同的 `config.yaml` 和 `plugins.yaml` 进行配置。



## Test-Infra 介绍

作为 kubernetes 基础保障，test-infra 功能非常的强大。

![img](https://raw.githubusercontent.com/kubernetes/test-infra/9771710c13868bddd1476170a77ddab36941c512/docs/architecture.svg)

**架构解释：**

对于 test-infra 的架构，我们首先可以发现，其相对复杂，并且包含许多微服务组件。值得注意的一点是，test-infra 中的微服务与其他微服务之间的交互并不采用我们熟知的传统方式，例如，它并不通过 grpc 进行调用，这与 OpenIM 等传统微服务有所区别。

test-infra 的架构的核心组成部分是 Hooks，它负责接收不同类型的事件。然后，test-infra 会通过一套插件系统，根据事件类型将其分发给不同的插件进行处理。例如，如果我们考虑一个实例，那么在 kubernetes test-infra 仓库的 prow 目录下，我们可以找到一个名为 plugins 的插件集合，其中一个插件名为 `transfer-issue`。此插件负责处理 PR 请求。

另一方面，对于单元请求，它将进行单元测试，对于合并请求，它将进行合并测试。在这个系统中，我们还有一个称为 `prowjob` 的CRD资源，它提供了一种高层次的抽象，以及一个自定义的控制器。

所有的测试结果会被回传到 GitHub 的测试面板上。状态更新会通过 crier 进行，然后传入到对应的组织和仓库地址。

在可视化方面，test-infra 提供了一个名为 deck 的组件。对应的网站是 prow.k8s.io，它提供了前端视图，使得用户可以更直观地理解和掌控整个测试流程。

这样的设计让 test-infra 的架构显得既复杂又灵活，但也带来了极高的定制性和扩展性。



**基本组件：**

1. Prow Controller Manager：Prow控制器管理器是Prow的核心组件，负责协调Prow的各个子系统。它监控Git存储库中的事件，并根据配置触发相应的操作。
2. Prow Job：Prow Job定义了CI/CD系统中的一个任务或作业。它描述了要运行的代码、测试和部署步骤，并指定了触发该作业的条件。
3. Prow Plugin：Prow插件是一种扩展机制，允许开发人员为Prow系统添加自定义功能。插件可以监听事件并执行相应的操作，例如自动化代码审查、生成报告等。
4. Prow Dashboard：Prow仪表板是一个Web界面，用于监视和管理Prow系统的运行状态。它提供了对作业、插件和事件的可视化界面，方便用户查看和操作。
5. Plank：Plank是Prow的任务调度器，负责将Prow Job分配给可用的工作节点进行执行。它会监控作业队列，并将作业分发给合适的工作节点，以便并行执行作业。
6. Hook：Hook是Prow的事件处理器，用于接收和处理来自Git存储库的事件。它会监听Git存储库中的事件，并将这些事件转发给Prow的Controller Manager进行处理。
7. Deck：Deck是Prow的用户界面，提供了一个Web界面，用于查看Prow系统中的作业、插件和事件等信息。开发人员可以使用Deck来监视和管理CI/CD流程，查看作业的状态和日志等。
8. Sinker：Sinker是Prow的清理器，负责清理过期的作业和资源。它会定期检查作业的状态，并清理已完成或过期的作业，以释放资源并保持系统的整洁。
9. Tide：Tide是Prow的自动合并管理器，用于管理代码合并流程。它会监视Git存储库中的Pull Request，并根据配置的规则自动合并符合条件的Pull Request。



### k8s prow 能支持哪些日志存储

**之前 kubesphere 中听过两句话：**

Prow  目前只是支持 Github ，Gerrit，对于 gitlab 的支持短期难以看到。

> 但是我们可以通过 `Lighthouse` 去支持

~~prow 的持久化存储只支持 GCP，但是可以使用 Jenkins X，它使用了 Knative 来跑 Job~~

> 这句话不对，我们可以通过 prow 找到，是可以支持其他的云的。Prow 并不仅用 GCP，只要是兼容 s3 的 sdk ，就可以。



### 基础满足

### 创建 access tokens

https://github.com/settings/tokens （*需要在 `.env` 里配置*）

### 创建 webhook

仓库的所有事件，都需要通过 webhook 去监听使用的。

https://github.com/用户名/项目名/settings/hooks/new

+ Payload URL: [www.example.com:8000](http://www.example.com:8000/)
+ Content type: application/json
+ trigger: Send me everything.
+ Secret: xxx （*需要在 .env 里配置*）



### 开发运行

```
npm install
cp env .env
vim .env
npm start
```



### 部署

本项目使用 [pm2](https://github.com/Unitech/pm2) 进行服务管理，发布前请先全局安装 [pm2](https://github.com/Unitech/pm2)

```
npm install pm2 -g
npm run deploy
```

后台启动该服务后，可以通过 `pm2 ls` 来查看服务名称为 `github-bot` 的运行状态。具体 [pm2](https://github.com/Unitech/pm2) 使用，请访问：https://github.com/Unitech/pm2



### 日志系统说明

本系统 `logger` 服务基于 [log4js](https://github.com/log4js-node/log4js-node)。 在根目录的 `.env` 文件中有个参数 `LOG_TYPE` 默认为 `console`，参数值说明：

```
console - 通过 console 输出log。
file - 将所有相关log输出到更根目录的 `log` 文件夹中。
```



## END 链接

### Link

+ [prow docs](https://docs.prow.k8s.io/docs/overview/)
+ [github kubernetes/test-infra/prow/cron](https://github.com/kubernetes/test-infra/blob/master/prow/cron/cron.go?rgh-link-date=2023-05-13T15%3A35%3A30Z)
+ [using commond](https://prow.k8s.io/command-help)
+ [@k8s-ci-robot](https://github.com/k8s-ci-robot)
+ [@ks-ci-bot](https://github.com/ks-ci-bot)



### 参考文章

+ [xuexb GitHub](https://github.com/xuexb/github-bot)
+ [sealos gh rebot](https://github.com/labring/gh-rebot)
+ https://www.amoyw.com/2020/10/22/Prow/



### Test Infra

包含 prow：

- Istio: https://github.com/istio/test-infra
- Kubernetes: https://github.com/kubernetes/test-infra
- Knative: https://github.com/knative/test-infra



不含有 prow：

- prometheus： https://github.com/prometheus/test-infra
- kyma-project： https://github.com/kyma-project/test-infra
- Grpc: https://github.com/grpc/test-infra



### 文档

[Prow: Keeping Kubernetes CI/CD Above Water - Kurt Madel](https://kurtmadel.com/posts/native-kubernetes-continuous-delivery/prow/)

[Prow, Jenkins X Pipeline Operator, and Tekton: Going Serverless With Jenkins X](https://www.alldaydevops.com/blog/prow-jenkins-x-pipeline-operator-and-tekton-going-serverless-with-jenkins-x)

[Jenkins X replaces Prow with Lighthouse for better source control compatibility • DEVCLASS](https://devclass.com/2020/06/18/jenkins-x-cloudbees-may-update/)

[Prow + Kubernetes - A Perfect Combination To Execute CI/CD At Scale](https://www.velotio.com/engineering-blog/prow-for-native-kubernetes-ci-cd)



#### 参考

[Overview](https://docs.prow.k8s.io/docs/overview/)



#### 代码部分：

[test-infra/prow at master · kubernetes/test-infra](https://github.com/kubernetes/test-infra/tree/master/prow#bots-home)