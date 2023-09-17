---
title: '一份完整的开源贡献指南（第一次踏入开源）'
description:
ShowRssButtonInSectionTermList: true
cover.image:
date : 2023-09-16T16:40:54+08:00
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



## 任务分配

> time：Within a week

- 完成 first contribute，目的：了解开源项目的贡献流程
- 完成 sealos 开发环境构建
- 了解 kuberentes 基本使用，核心概念，核心组件的作用

> - 基本使用：
>   - 创建 一个 pod 并理解什么是 pod
>   - 创建一个 deployment 理解 deployment 与 pod 的关系
>   - 创建一个 configmap， 理解挂载配置文件给 pod
>   - 创建一个 service，通过 service 在集群内访问 pod
> - 核心概念，核心组件的作用：
>   - kubectl apiserver controller-manager scheduler kubelet kube-proxy etcd 这些组件分别是做什么的
>   - 可以用一个 kubectl apply 一个 deployment 这些组件分别做了哪些事来梳理整个流程

🚸 next time：会分配一个具体的任务以及介绍 sealos 源码架构。



### 资源🗓️

**参考资料：**

1. 贡献文档：[https://github.com/labring/sealos/blob/main/CONTRIBUTING.md](https://github.com/labring/sealos/blob/main/CONTRIBUTING.md)

2. 开发环境搭建文档：[https://github.com/labring/sealos/blob/main/DEVELOPGUIDE.md](https://github.com/labring/sealos/blob/main/DEVELOPGUIDE.md)

3. 使用 sealos 快速构建 kubernetes 学习环境文档：[https://github.com/labring/sealos#quickstart](https://github.com/labring/sealos#quickstart) 搭建单机环境即可。

4. kubernetes 入门文档：[https://kubernetes.io/docs/tutorials/kubernetes-basics/](https://kubernetes.io/docs/tutorials/kubernetes-basics/)  安装部分跳过直接使用 sealos 一键构建。


> ⚠️ 注：中间出现任何问题一定不要怕骚扰 fanux，主动提问题很重要。孵化交流群里也非常欢迎问问题。



## 贡献文档

+ [x] [贡献指南](https://github.com/labring/sealos/blob/main/CONTRIBUTING.md)

对于发现的**安全**问题，建议使用发送邮箱的方式告知[admin@sealyun.com](mailto:admin@sealyun.com) 

对于**一般** 的问题，或许你可以选择 [issues]([New Issue · labring/sealos (github.com)](https://github.com/labring/sealos/issues/new/choose)) 来指出问题

![image-20221019161049208](http://sm.nsddd.top/smimage-20221019161049208.png)

⚡ 显然，相比较`issues`，我更喜欢`pr` ，你可以发现下面的问题并且改进

+ 如果您发现拼写错误，请尝试修复它！
+ 如果您发现错误，请尝试修复它！
+ 如果您发现一些多余的代码，请尝试删除它们！
+ 如果发现缺少一些测试用例，请尝试添加它们！
+ 如果您可以增强功能，**请不要犹豫**！
+ 如果您发现代码是隐式的，请尝试添加注释以使其清晰！
+ 如果你觉得代码很丑陋，试着重构它！
+ 如果你能帮助改进文档，那就再好不过了！
+ 如果您发现文档不正确，请直接进行修复！
+ .…..



### 🧷 补充阅读

1. [如何参与github项目或许你可以参考这篇文章~](https://nsddd.top/archives/contributors)
2. [如何使用actions自动部署实现自动更新远程~](https://nsddd.top/archives/actions)

### 💡 步骤

+ [x] [git的教程](https://github.com/cubxxw/awesome-cs-course/blob/master/Git/README.md)

⬇️ 大致流程如下：

1. 首先在Github上fork本仓库到你的仓库

2. git clone克隆到本地
3. 在本地修改对应的代码
4. git push到自己的仓库
5. 在自己的仓库进行pull request的操作



### 文档规范

#### 格式

请遵守以下规则以更好地格式化文档，这将大大改善阅读体验。

1. 请不要在英文文档中使用中文标点符号，反之亦然。
2. 请在适用的情况下使用大写字母，例如句子/标题的第一个字母等。
3. 请为每个 Markdown 代码块指定一种语言，除非没有关联的语言。
4. 请在中文和英文单词之间插入空格。
5. 请使用正确的技术术语大小写，例如使用 HTTP 而不是 http，使用 MySQL 而不是 mysql，使用 Kubernetes 而不是 kubernetes 等。
6. 请在提交 PR 之前检查文档中是否有任何拼写错误。

您还可以查看[docusaurus](https://docusaurus.io/docs/markdown-features)，以编写具有更丰富功能的文档。



**1. 将“远程上游”设置为**使用以下两个命令：`https://github.com/labring/sealos.git`

> [🧷 git添加远程仓库的两种方式](https://github.com/cubxxw/awesome-cs-course/blob/master/Git/markdown/git-adds.md)

```bash
git remote add upstream https://github.com/labring/sealos.git
git remote set-url --push upstream no-pushing
```

![image-20221109173951312](http://sm.nsddd.top/smimage-20221109173951312.png)

使用此远程设置，您可以像这样检查 git 远程配置：

```bash
$ git remote -v
origin     https://github.com/<your-username>/sealos.git (fetch)
origin     https://github.com/<your-username>/sealos.git (push)
upstream   https://github.com/labring/sealos.git (fetch)
upstream   no-pushing (push)
```

添加此内容，我们可以轻松地将本地分支与上游分支同步。

![image-20221019162733226](http://sm.nsddd.top/smimage-20221019162733226.png)



**2. 创建分支**以添加新功能或修复问题

更新本地工作目录和远程分叉存储库：

> 推荐使用`fetch`：从安全角度出发，`git fetch`比`git pull`更安全，因为我们可以先比较本地与远程的区别后，选择性的合并。

```bash
cd sealos
git fetch upstream
git checkout main
git rebase upstream/main
git push	# default origin, update your forked repository
```

创建新分支：

```
git checkout -b bug-xiongxinwei
```

对然后构建和测试代码进行任何更改。`new-branch`

> 推荐的命名规则：
>
> ```asciiarmor
> 分支:		命名:		说明:
> 
> 主分支		master		主分支，所有提供给用户使用的正式版本，都在这个主分支上发布
> 开发分支		dev 		开发分支，永远是功能最新最全的分支
> 功能分支		feature-*	新功能分支，某个功能点正在开发阶段
> 发布版本		release-*	发布定期要上线的功能
> 修复分支		bug-*		修复线上代码的 bug
> ```

![image-20221019164941695](http://sm.nsddd.top/smimage-20221019164941695.png)



**3. 将分支推送**到分叉的存储库，尽量不要在 pr 中生成多个提交消息。

> `git commit -a -s -m "message for your changes"`
>
> + `-a` 参数设置修改文件后不需要执行 `git add` 命令，直接来提交
> + `-s` 表示添加了一个签名，加入了自己的信息
>
> ![image-20221019190552361](http://sm.nsddd.top/smimage-20221019190552361.png)

```bash
golangci-lint run -c .golangci.yml # lint
git add -A
git commit -a -s -m "message for your changes" # -a is git add ., -s adds a Signed-off-by trailer
git rebase -i	<commit-id>  # 如果你的pr有多次提交
git push   # 在rebase完成后推送到分叉库，如果是第一次推送，运行git push --set-upstream origin <new-branch>
```

![image-20221019190409127](http://sm.nsddd.top/smimage-20221019190409127.png)

> 为每个 Markdown 代码块指定一种语言，除非没有关联的语言。

如果不想使用 ，可以使用`git rebase -i`、`git commit -s --amend && git push -f`

如果在同一分支中开发多个功能，则应重定主分支的基调：

```bash
# create new branch, for example git checkout -b feature/infra
git checkout -b <new branch>
# update some code, feature1
git add -A
git commit -m -s "init infra"
git push # if it's first time push, run git push --set-upstream origin <new-branch>
# then create pull request, and merge
# update some new feature, feature2, rebase main branch first.
git rebase upstream/main
git commit -m -s "init infra"
# then create pull request, and merge
```



**提交拉取请求给主分支：**

![image-20221019192522791](http://sm.nsddd.top/smimage-20221019192522791.png)



## 使用 sealos 快速构建 kubernetes

> ⚠️ 安装注意事项：
>
> 1. 需要**纯净版**Linux系统 `ubuntu16.04`，`centos7`
> 2. 版本使用新版，**用新不用旧**
> 3. 必须同步服务器时间
> 4. 主机名不可重复
> 5. master结点CPU必须2C以上
> 6. cni组件选择`cilium`时要求内核版本不低于5.4

+ [x] [快速搭建指南](https://github.com/labring/sealos/blob/main/DEVELOPGUIDE.md)

`sealos`现在只支持`linux`，需要`linux`服务器来测试。

一些工具可以非常方便地帮助您启动虚拟机，例如[multipass](https://multipass.run/)



### 构建项目

```bash
mkdir /sealos && cd /sealos ; git clone https://ghproxy.com/https://github.com/labring/sealos && cd sealos && ls ; make build  # 大概可能因为网络原因需要等一段时间~
```

您可以将 `bin` 文件 `scp` 到您的 `linux` 主机。

如果你使用 `multipaas`，你可以将 `bin` 目录挂载到 `vm`：

```bash
multipass mount /your-bin-dir <name>[:<path>]
```

然后在本地测试。

> **⚠️ 注意：**
>
> 所有二进制文件都`sealos`可以在任何地方构建，因为它们有`CGO_ENABLED=0`. 但是，在运行一些依赖于 CGO 的`sealos`子命令时，需要支持覆盖驱动程序。`images`因此，在构建时会打开 CGO `sealos`，从而无法`sealos`在 Linux 以外的平台上构建二进制文件。
>
> + 本项目中的 `Makefile` 和 `GoReleaser` 都有这个设置。
>
> ## Install golang
>
> ```bash
> wget -o https://go.dev/dl/go1.19.3.linux-amd64.tar.gz && tar -C /usr/local -zxvf go1.19.3.linux-amd64.tar.gz
> cat >> /etc/profile <<EOF
> # set go path
> export PATH=\$PATH:/usr/local/go/bin
> EOF
> source /etc/profile  && go version
> ```
>
> ## Build the project
>
> ```bash
> git clone https://github.com/labring/sealos && cd sealos
> go env -w GOPROXY=https://goproxy.cn,direct && make build
> ```

::: danger 自定义环境变量mypath
root@smile:/usr/local/src# cat /etc/profile.d/mypath

# GO语言路径

export GO_PATH=$"/usr/local/src/go"

# path

export PATH=$PATH:$GO_PATH/bin

:::
---

😂 让我很喜欢的一点是：`sealos`能一次性把环境搭建好，想当年，我真是废了九牛二虎之力才搭建~失败的。

![image-20221019194939030](http://sm.nsddd.top/smimage-20221019194939030.png)



### 远程连接

+ [x] [远程连接 & 免密远程~文档](https://github.com/cubxxw/awesome-cs-course/blob/master/linux/linux-web/7.md)



### 遇到的坑和解决方案

+ centos服务器不推荐使用（建议使用ubuntu)
+ go版本最好`>18`



::: tip 

1. 克隆代码慢，你可以使用 ghproxy：`git clone https://ghproxy.com/https://github.com/labring/sealos`
2. 构建下载包慢，可以使用 **goproxy** ：`go env -w GOPROXY=https://goproxy.cn,direct && make build`
3. `cgo: C compiler "x86_64-linux-gnu-gcc" not found: exec: "x86_64-linux-gnu-gcc": executable file not found in $PATH`您需要安装 gnu-gcc，例如：`apt-get install build-essential`或`yum -y install gcc-c++-x86_64-linux-gnu`

:::



## 使用 sealos 快速构建 kubernetes

+ [x] [sealos构建k8s指南](https://github.com/labring/sealos#quickstart)



### 添加到环境变量

> `sealos`位置`/sealos/*`

```bash
export PATH=$PATH:/sealos/bin/linux_amd64/
#export PATH=/usr/local/bin:$PATH
// PATH是变量名，这里是指添加到PATH这个环境变量中
// =后面是要添加的环境变量
// :$PATH是指把新添加的环境变量与原先的环境变量重新赋值给PATH这个变量，这里可以看出如果有多个环境变量时，应该使用:进行分隔，如
// export PATH=/sealos/bin/linux_amd64/bin:/sealos/bin/linux_amd64/bin:$PATH
// 当然$PATH是放在开头还是最后是没有影响的
```

**这种方法添加的环境变量会立即生效，但是在窗口关闭后便会失效**

⬇️ 添加全局变量方法：

```bash
vim /etc/profile
// 如果只修改当前用户的环境变量，则是`vim ~/.bashrc`
// 在文件的最后一行添加以下代码：
export PATH=$PATH:/sealos/bin/linux_amd64/
// 规则和用法如第二条所说
```

⚔️ 快捷：

```bash
cat >> /etc/profile <<EOF
# set go path
export PATH=\$PATH:/usr/local/go/bin
EOF

echo "source /etc/profile" >> ~/.bashrc  #auto update  
```

 

⚡验证：

```bash
root@VM-4-3-ubuntu:/# sealos version
{"gitVersion":"untagged","gitCommit":"b24684f6","buildDate":"2022-10-20T19:20:05+0800","goVersion":"go1.19.2","compiler":"gc","platform":"linux/amd64"}
root@VM-4-3-ubuntu:/# 
```



## k8s入门文档

+ [x] [官方入门文档](https://kubernetes.io/docs/tutorials/kubernetes-basics/) 



### docker、k8s、云原生笔记

+ [x] [docker.nsddd.top](https://docker.nsddd.top)



### 任务块

- 基本使用：

  - 创建一个 `pod` 并理解什么是 `pod`   ➡️  [🧷记录](https://docker.nsddd.top/Cloud-Native-k8s/9.html)
  - 创建一个 `deployment` 理解 `deployment` 与 `pod` 的关系  ➡️  [🧷记录](https://docker.nsddd.top/Cloud-Native-k8s/10.html)
  - 创建一个 `configmap`， 理解挂载配置文件给 `pod`  ➡️  [🧷记录](https://docker.nsddd.top/Cloud-Native-k8s/13.html)
  - 创建一个 `service`，通过 `service` 在集群内访问 `pod`  ➡️  [🧷记录](https://docker.nsddd.top/Cloud-Native-k8s/11.html)

- 核心概念，核心组件的作用：

  > `Kube-proxy`负责制定数据包的转发策略，并以守护进程的模式对各个节点的`pod`信息实时监控并更新转发规则，`service`收到请求后会根据`kube-proxy`制定好的策略来进行请求的转发，从而实现负载均衡。

  可以用一个 `kubectl apply` 一个 `deployment` 这些组件分别做了哪些事来梳理整个流程



::: details 核心组件
`kubectl apiserver controller-manager scheduler kubelet kube-proxy etcd`  

>这些组件分别是做什么的?

+ 可以想象sealos很久之后形成的一个集团😂 
+ 我们有很多的厂，master主厂，node小厂

> Node节点主要包括kubelet、kube-proxy模块和pod对象
> Master节点主要包括API Server、Scheduler、Controller manager、etcd几大组件

+ kubectl 是 Kubernetes 自带的客户端，可以用它来直接操作 Kubernetes 集群。
+ Api Server相当于master的秘书，master和node的所有通信都需要走Api Server
+ controller-manager就是老大，是公司的决策者，负责集群内 Node、Namespace、Service、Token、Replication 等资源对象的管理，使集群内的资源对象维持在预期的工作状态。
+ scheduler就是调度者，如果我们小厂有东西做不出来那么需要scheduler负责对集群内部的资源进行调度，相当于“调度室”。
+ kubelet 就是小厂的厂长，对每个node进行操控
  + kubelet 组件通过 api-server 提供的接口监测到 kube-scheduler 产生的 pod 绑定事件，然后从 etcd 获取 pod 清单，下载镜像并启动容器。
  + 同时监视分配给该Node节点的 pods，周期性获取容器状态，再通过api-server通知各个组件。
+ kube-proxy这个就好理解了，就相当于sealos下面每个厂的门卫大爷，集团可能不知道哪个资源在哪个厂，但是门卫大爷肯定知道啊，所以各个node的kube-proxy是相通的。

:::



**what is `pod`？**

+ [🧷 Go to cub to learn pod ](https://docker.nsddd.top/Cloud-Native-k8s/9.html#%E4%BF%AE%E6%94%B9pod)

Pod is the smallest scheduling unit in `Kubernetes`. A Pod encapsulates a container (or multiple containers). Containers in a Pod share storage, network, etc. That is, you can think of the entire pod as a virtual machine, and then each container is equivalent to a process running on the virtual machine. All containers in the same pod are scheduled and scheduled uniformly.

> 在`Kubernetes`中部署应用时，都是以`pod`进行调度的，它们基本上是单个容器的包装或房子。从某种意义上说，容器的容器。 `pod`是一个逻辑包装实体，用于在`K8s`集群上执行容器。可以把每个`pod`想象成一个透明的包装，为容器提供一个插槽。`pod`是`Kubernetes`最小的可部署单位。`pod`是一组一个或多个容器，具有共享的存储/网络资源，以及如何运行容器的规范。因此，最简单地说，`pod`是一个容器如何在`Kubernetes`中“用起来”的机制。
>
> 1. pod是k8s的最小单元，容器包含在pod中，一个pod中有一个pause容器和若干个业务容器，而容器是单独的一个容器，简而言之，pod是一组容器的集合。
>
> 2. pod相当于逻辑主机，每个pod都有自己的ip地址
>
> 3. **pod内的容器共享相同的ip和端口**
>
> 4. 默认情况下，每个容器的文件系统与其他容器完全隔离

```bash
#创建一个名为nginx-learn的pod，暴露容器端口为80.
root@VM-4-3-ubuntu:/# kubectl get node -A
root@VM-4-3-ubuntu:/# kubectl run nginx-learn  --image=nginx:latest --image-pull-policy='IfNotPresent'  --port=80
pod/nginx-learn created
root@VM-4-3-ubuntu:/# kubectl get node -A
NAME            STATUS   ROLES           AGE   VERSION
vm-4-3-ubuntu   Ready    control-plane   24h   v1.25.0
root@VM-4-3-ubuntu:/# kubectl get pod
NAME          READY   STATUS    RESTARTS   AGE
nginx-learn   1/1     Running   0          58s
```



**delect the pod：**

```bash
root@VM-4-3-ubuntu:/# kubectl get pod
NAME          READY   STATUS    RESTARTS   AGE
nginx-learn   1/1     Running   0          58s
root@VM-4-3-ubuntu:/# kubectl delete pod nginx-learn
pod "nginx-learn" deleted
root@VM-4-3-ubuntu:/# kubectl get pod
No resources found in default namespace.
```



**kubectl创建和删除一个pod相关操作：**

| **命令** | **说明**                              |
| -------- | ------------------------------------- |
| run      | 在集群上运行一个pod                   |
| create   | 使用文件或者标准输入的方式创建一个pod |
| delete   | 使用文件或者标准输入来删除某个pod     |



**create deployment：**

> Pod是单一亦或一组容器的合集
>
> Pod是k8s的最小调度单位，一个Pod中可以有多个containers，彼此共享网络等，这是k8s的核心概念。
>
> **deployment是pod版本管理的工具 用来区分不同版本的pod**
>
> 从开发者角度看，deployment顾明思意，既部署，对于完整的应用部署流程，除了运行代码(既pod)之外，需要考虑更新策略，副本数量，回滚，重启等步骤，
>
> deployment，StatefulSet是Controller，保证Pod一直运行在你需要的状态。
>
> 有一次性的也就是job，有定时执行的也就是crontabjob，有排号的也就是sts

```bash
kubectl run nginx --image=nginx --replicas=2
```

> [nginx](https://so.csdn.net/so/search?q=nginx&spm=1001.2101.3001.7020)：应用名称
>
> `--replicas`：指定应用运行的 pod 副本数
>
> `--image`：使用的镜像（默认从dockerhub拉取）

```
kubectl get deployment 或者 kubectl get deploy
```



**查看 replicaset：**

```bash
kubectl get replicaset 或者 kubectl get rs
```


 **查看 pod：**

```bash
kubectl get pods -o wide
```



**create configMap：**

可以使用 `kubectl create configmap` 或者在 `kustomization.yaml` 中的 ConfigMap 生成器来创建 ConfigMap

```powershell
kubectl create configmap <映射名称> <数据源>
```

其中，`<映射名称>` 是为 ConfigMap 指定的名称，`<数据源>` 是要从中提取数据的目录、 文件或者字面值。ConfigMap 对象的名称必须是合法的 [DNS 子域名](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/names#dns-subdomain-names)。





### 多结点

sealos现在只支持linux，需要linux服务器来测试。

一些工具可以非常方便地帮助您启动虚拟机，例如[multipass](https://multipass.run/)

### 构建项目

```bash
mkdir /sealos && cd /sealos && git clone https://github.com/labring/sealos && cd sealos && ls && make build  # 大概可能因为网络原因需要等一段时间~
```

您可以将 `bin` 文件 `scp` 到您的 `linux` 主机。

如果你使用 `multipaas`，你可以将 `bin` 目录挂载到 `vm`：

```bash
multipass mount /your-bin-dir <name>[:<path>]
```

然后在本地测试。

> **⚠️ 注意：**
>
> 所有二进制文件都`sealos`可以在任何地方构建，因为它们有`CGO_ENABLED=0`. 但是，在运行一些依赖于 CGO 的`sealos`子命令时，需要支持覆盖驱动程序。`images`因此，在构建时会打开 CGO `sealos`，从而无法`sealos`在 Linux 以外的平台上构建二进制文件。
>
> + 本项目中的 `Makefile` 和 `GoReleaser` 都有这个设置。

---

😂 让我很喜欢的一点是：`sealos`能一次性把环境搭建好，想当年，我真是废了九牛二虎之力才搭建~失败的。

![image-20221019194939030](http://sm.nsddd.top/smimage-20221019194939030.png)



## 核心服务快速启动

**💡 重新把昨天集群全部删除，新开三台服务器，纯新~**

![image-20221021151347038](http://sm.nsddd.top/smimage-20221021151347038.png)



### 环境准备

> ⚠️ 注意：环境一定很重要，不然都跑不起来~

```bash
hostnamectl set-hostname k8s-master01
hostnamectl set-hostname k8s-master02
hostnamectl set-hostname k8s-master03
```

> **虚拟机需要配置静态IP**



### 查看内核版本

```bash
# 下载并安装 sealos, sealos 是个 golang 的二进制工具，直接下载拷贝到 bin 目录即可, release 页面也可下载 
yum install wget && yum install tar &&\
wget  https://github.com/labring/sealos/releases/download/v4.1.3/sealos_4.1.3_linux_amd64.tar.gz  && \
tar -zxvf sealos_4.1.3_linux_amd64.tar.gz sealos &&  chmod +x sealos && mv sealos /usr/bin 
# 创建一个集群
sealos run labring/kubernetes:v1.25.0 labring/helm:v3.8.2 labring/calico:v3.24.1 \
     --masters 192.168.0.2,192.168.0.3\
     --nodes 192.168.0.4 -p [your-ssh-passwd]
```

> `-p`：passwd密码
>
> 开启ssh免密不需要些密码了，在这里就实现了。
>
> ![image-20221020111912006](http://sm.nsddd.top/smimage-20221020111912006.png)

![image-20221020105230320](http://sm.nsddd.top/smimage-20221020105230320.png)



**验证集群：**

```bash
kubectl get nodes
```

![image-20221020113615770](http://sm.nsddd.top/smimage-20221020113615770.png)



### 单节点

> Single host
>
> + You can use `multipass` to start multiple virtual machines from one machine
> + Recent releases also support the ——`Single` mode single-machine deployment

```bash
$ sealos run labring/kubernetes:v1.25.0 labring/helm:v3.8.2 labring/calico:v3.24.1 --single
# remove taint
$ kubectl taint node --all node-role.kubernetes.io/control-plane-
```

![image-20221020212025716](http://sm.nsddd.top/smimage-20221020212025716.png)