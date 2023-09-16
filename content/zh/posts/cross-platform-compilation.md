---
title : '跨平台以及多架构编译设计'
description:
ShowRssButtonInSectionTermList: true
cover.image:
date : 2023-02-13T16:21:53+08:00
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



## 前言

https://github.com/OpenIMSDK/Open-IM-Server/issues/432

现在很多地方都对服务的国产化适配有所要求，一般的国产化平台都提供arm版本的linux云环境供我们进行服务部署，因此需要构建arm版本的镜像。

## 构建方案

在上面的 issue 中我们描述了大致的构建思路和解决的步骤，我们来看一下构建的方案，我们以最常用的 amd 机器为例，来编译 arm。对于构建镜像的ARM版本，有如下两种方式：

1. 在ARM机器上使用 docker build 进行构建；
2. 在X86/AMD64 的机器上使用 docker buildx 进行交叉构建；

> **⚠️注意：**
>
> 1. 交叉构建和交叉运行的方式会有一些无法预知的问题，建议简单的构建步骤（如只是下载解压对应架构的文件）可考虑在x86下交叉构建，复杂的（如需要编译的）则直接在arm机器上进行构建；
> 2. 实际测试发现，使用[qemu方式](https://github.com/multiarch/qemu-user-static)在x86平台下运行arm版本的镜像时，执行简单的命令可以成功（如arch），执行某些复杂的程序时（如启动java虚拟机），会无响应，所以镜像的验证工作应尽量放置到arm机器上进行；
>
> **上面第二点按如下方式测试：**
>
> + `docker run --rm --platform=linux/arm64 openjdk:8u212-jre-alpine arch` 可正常输出；
> + `docker run --rm --platform=linux/arm64 openjdk:8u212-jre-alpine java -version` 则会 **卡住**，且需要使用`docker stop`停止容器才可以退出容器；

## 启用试验性功能

> 💡 注意：buildx 仅支持 docker19.03 及以上docker版本

如需使用 buildx，需要开启docker的实验功能后，才可以使用，开启方式：

编辑 `/etc/docker/daemon.json` ，添加：

```jsx
{
    "experimental": true
}
```

编辑 `～/.docker/config.json` 添加：

```jsx
"experimental" : "enabled"
```

重启Docker使生效：

+ `sudo systemctl daemon-reload`
+ `sudo systemctl restart docker`

确认是否开启：

+ `docker version -f'{{.Server.Experimental}}'`
+ 如果输出true，则表示开启成功

在之前的版本中构建多种系统架构支持的 Docker 镜像，要想使用统一的名字必须使用 `[$ docker manifest](notion://www.notion.so/docker_practice/image/manifest)` 命令。

在 Docker 19.03+ 版本中可以使用 `$ docker buildx build` 命令使用 `BuildKit` 构建镜像。该命令支持 `--platform` 参数可以同时构建支持多种系统架构的 Docker 镜像，大大简化了构建步骤。

## 使用buildx构建

buildx 的详细使用可参考：[Docker官方文档-Reference-buildx](https://docs.docker.com/engine/reference/commandline/buildx/?fileGuid=0l3NVKX0BgflYN3R)

### **创建 buildx 构建器**

使用 docker buildx ls 命令查看现有的构建器

```bash
root@rbqntnwlflfxvigv:~# docker buildx ls
NAME/NODE DRIVER/ENDPOINT STATUS  BUILDKIT PLATFORMS
default * docker                           
  default default         running 20.10.24 linux/amd64, linux/386
```

创建并构建器：

```bash
# 下面的创建命令任选一条符合情况的即可
# 1. 不指定任何参数创建
docker buildx create --use --name multiarch-builder
# 2. 如创建后使用docker buildx ls 发现构建起没有arm架构支持，可使用--platform明确指定要支持的构建类型，如以下命令
docker buildx create --platform linux/arm64,linux/arm/v7,linux/arm/v6 --name multiarch-builder
# 3. 如需在buildx访问私有registry，可使用host模式，并手动指定配置文件，避免buildx时无法访问本地的registry主机
docker buildx create --platform linux/amd64,linux/arm64,linux/arm/v7,linux/arm/v6  --driver-opt network=host --config=/Users/hanlyjiang/.docker/buildx-config.toml --use --name multiarch-builder
```

buildx-config.toml 配置文件写法类似：

```bash
# <https://github.com/moby/buildkit/blob/master/docs/buildkitd.toml.md>
# registry configures a new Docker register used for cache import or output.
[registry."zh-registry.geostar.com.cn"]
  mirrors = ["zh-registry.geostar.com.cn"]
  http = true
  insecure = true
```

**启用构建器**

```bash
# 初始化并激活
docker buildx inspect multiarch-builder --bootstrap
```

**确认成功**

```bash
# 使用 docker buildx ls 查看
docker buildx ls
```

Docker 在 Linux 系统架构下是不支持 arm 架构镜像，因此我们可以运行一个新的容器让其支持该特性，Docker 桌面版则无需进行此项设置（mac系统）。

+ 在内核中使用 QEMU 仿真支持来进行多架构镜像构建

```bash
# 安装模拟器（用于多平台镜像构建）
$ docker run --rm --privileged tonistiigi/binfmt:latest --install all
```

由于 Docker 默认的 `builder` 实例不支持同时指定多个 `--platform`，我们必须首先创建一个新的 `builder` 实例。同时由于国内拉取镜像较缓慢，我们可以使用配置了 [镜像加速地址](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fmoby%2Fbuildkit%2Fblob%2Fmaster%2Fdocs%2Fbuildkitd.toml.md) `[dockerpracticesig/buildkit:master](<https://github.com/docker-practice/buildx>)` 镜像替换官方镜像

```
# 适用于国内环境
root@i-3uavns2y:~# docker buildx create --use --name=mybuilder-cn --driver docker-container --driver-opt image=dockerpracticesig/buildkit:master

# 适用于腾讯云环境(腾讯云主机、coding.net 持续集成)
root@i-3uavns2y:~# docker buildx create --use --name=mybuilder-cn --driver docker-container --driver-opt image=dockerpracticesig/buildkit:master-tencent
# 使用默认镜像
root@i-3uavns2y:~# docker buildx create --name mybuilder --driver docker-container

# 使用新创建好的 builder 实例
root@i-3uavns2y:~# docker buildx use mybuilder
```

查看已有的 builder 实例

```go
root@i-tpmja312:~# docker buildx ls
NAME/NODE    DRIVER/ENDPOINT             STATUS   PLATFORMS
mybuilder *  docker-container
  mybuilder0 unix:///var/run/docker.sock inactivedefault      docker
  default    default                     running  linux/amd64, linux/386
```

构建：

```yaml
docker buildx build --platform linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/386,linux/ppc64le,linux/s390x -t kubecub/hello . --push
```

### **修改Dockerfile**

对 Dockerfile 的修改，大致需要进行如下操作：

1. 确认基础镜像（FROM）是否有arm版本，如果有，则可以不用改动，如果没有，则需要寻找替代镜像，如没有替代镜像，则可能需要自行编译；
2. 确认dockerfile的各个步骤中是否有依赖CPU架构的，如果有，则需要替换成arm架构的，如在构建jitis的镜像时，Dockerfile中有添加一个amd64架构的软件

```
ADD <https://github.com/just-containers/s6-overlay/releases/download/v1.21.4.0/s6-overlay-amd64.tar.gz> /tmp/s6-overlay.tar.gz
```

此时需要替换为下面的地址(注意amd64替换成了aarch64，当然，需要先确认下载地址中有无对应架构的gz包，不能简单做字符替换)：

```
ADD <https://github.com/just-containers/s6-overlay/releases/download/v1.21.4.0/s6-overlay-aarch64.tar.gz> /tmp/s6-overlay.tar.gz
```

当然，我们需要确认该软件有此架构的归档包，如果没有，则需要考虑从源码构建；

> 提示：
>
> 怎么确定一个可执行文件 `/so` 库的对应的执行架构？ 可以通过 `file {可执行文件路径}` 来查看，
>
> 如下面时macOS上执行file命令的输入，可以发现macOS上的git程序可以兼容两种架构-`x86_64&arm64e`：
>
> ```
> file $(which git)
> /usr/bin/git: Mach-O universal binary with 2 architectures: [x86_64:Mach-O 64-bit executable x86_64] [arm64e:Mach-O 64-bit executable arm64e]
> /usr/bin/git (for architecture x86_64):	Mach-O 64-bit executable x86_64
> /usr/bin/git (for architecture arm64e):	Mach-O 64-bit executable arm64e
> ```
>
> 下面的命令则对一个so文件执行了file，可以看到其中的架构信息 `ARM aarch64`：
>
> ```
> file /lib/aarch64-linux-gnu/libpthread-2.23.so
> /lib/aarch64-linux-gnu/libpthread-2.23.so: ELF 64-bit LSB shared object, ARM aarch64, version 1 (GNU/Linux), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, BuildID[sha1]=880365ebb22114e4c10108b73243144d5fa315dc, for GNU/Linux 3.7.0, not stripped
> ```

### docker buildx 构建arm64镜像的命令

使用 --platform来指定架构，使用 `--push` 或 `--load` 来指定构建完毕后的动作。

```
docker buildx build --platform=linux/arm64,linux/amd64 -t xxxx:tag . --push
```

> 提示：当指定多个架构时，只能使用 --push 推送到远程仓库，无法 `--load`，推送成功后再通过 `docker pull --platform` 来拉取指定架构的镜像

### **检查构建成果**

1. 通过 `docker buildx imagetools inspect` 命令查看镜像信息，看是否有对应的arm架构信息；
2. 实际运行镜像，确认运行正常；（在arm机器上执行）

> 提示：如运行时输出 `exec format error` 类似错误，则表示镜像中部分可执行文件架构不匹配。

## **在x86上运行arm镜像**

可参考 [github/qemu-user-static](https://github.com/multiarch/qemu-user-static) ,简要描述如下：

+ 执行如下命令安装：

  `docker run --rm --privileged multiarch/qemu-user-static --reset -p yes`

+ 之后即可运行arm版本的镜像，如：

  ```
  docker run --rm -t arm64v8/fedora uname -m
  ```

## **在x86平台下使用Buildx构建跨平台镜像并运行arm应用**

我们演示了一下简单的构建方法，

### 安装 qemu 多平台支持

运行以下容器：

```
docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```

该容器会为你的设备安装 qemu 多平台支持，如果你需要运行跨平台容器，也会用到它。

### **创建新的 builder 实例并设为默认**

```
docker buildx create --use --name mybuilder
```

看到输出 `mybuilder` 即表示创建成功，使用 `--use` 指令将在 builder 实例创建完成时自动将其设为默认，否则需要手动使用 `docker buildx use mybuilder` 将创建的实例设为默认。

### **使用 Buildx 构建多平台镜像**

Buildx 的使用与 docker build 十分相似，基本上只需要将命令中的 `docker build` 替换成 `docker buildx build` 即可。如果使用 `docker buildx install` 将默认的 docker build 替换为 Buildx，那么直接使用 `docker build` 即可。

例如，将当前目录下的 Dockerfile 文件打包成镜像，需要使用以下命令：

```
docker buildx build -t xxx/xxx:tag . --push
```

如果替换了默认 docker build，将是这样的：

```
docker build -t xxx/xxx:tag . --push
```

`-push` 指令会自动把构建好的镜像推送到远端仓库，否则只会在存放在 cache 中。

如果要构建多平台镜像，在指令中加入 `--platform=` 即可，等号后填写需要构建的平台，如 `linux/arm`，`linux/arm64`，`linux/amd64` 等，用 `,` 隔开。Dockerfile 本身并不需要做出更改，除非你需要做的操作在不同平台下有所区别，比如根据平台下载不同文件等。

```docker
docker buildx build --platform=linux/arm,linux/arm64,linux/amd64 -t xxx/xxx:tag . --push
```

Buildx 将根据以上指令自动构建三个平台的镜像并推送到远端，这三个镜像会使用命令中指定的同一个 tag。

需要注意的是，指定的平台必须是底层镜像所支持的。

本地支持七种构建：

```yaml
docker buildx build --platform linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/386,linux/ppc64le,linux/s390x -t doubledong/hello . --push
```

在 [Docker Documents](https://docs.docker.com/buildx/working-with-buildx/) 查看更多详细的说明。

## 使用 GitHub Action 自动构建多平台镜像

由于 DockerHub 的自动构建工具对多平台支持并不友好，推荐使用 GitHub Action 来构建。具体的 yaml  文件如下：

```yaml
name: docker build and push

on:
  release:
    branches: [ main ]
    types: released
    # 将在main分支的release发布时自动运行该流程
  workflow_dispatch:
    # 将在GitHub Action界面创建一个run workflow按钮，点击后执行该流程
jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Get the tag name
        run: echo "TAG=${GITHUB_REF/refs\\/tags\\//}" >> $GITHUB_ENV
        # 获取release tag，在创建镜像时会用到
      - name: Setup QEMU
        uses: docker/setup-qemu-action@v1

      - name: Docker Setup Buildx
        uses: docker/setup-buildx-action@v1.3.0
        # 启用Buildx
      - name: Login
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
            # 登陆DockerHub账号，供推送镜像使用，这里的secrets需要在仓库设置页面添加
      - name: Build and Push with Version Tag
        uses: docker/build-push-action@v2
        with:
          context: .
          platforms: linux/amd64,linux/arm64,linux/arm
          push: true
          tags: xxx/abc:${{ env.TAG }}
          # 使用仓库根目录中的Dockerfile构建三个平台的镜像，并推送到xxx/abc仓库，使用之前获取的tag
```

### 跨平台运行容器的策略

我们知道了如何跨平台编译，那么拉取镜像的策略是什么样的呢？

一般来说。默认使用 docker pull 指令只会拉取和当前平台一致的镜像，要拉取其他平台的镜像，使用–platform 指定对应的平台。

同样，在使用 docker run 运行容器时也需要使用–platform 指定平台。

如果使用 docker-compose 来管理容器，需要在 image 的同级添加类似 `platform: linux/arm` 的指令来指定平台。如果本地已有相同 tag 的其他平台镜像，需要使用 `docker-compose pull` 来拉取需要平台的镜像

### 案例演示

假设有一个简单的 golang 程序源码：

```yaml
❯ cat hello.go
/*************************************************************************
   > File Name: hello.go
   > Author: smile
   > Mail: 3293172751nss@gmail.com
   > Created Time: Sun Jun 11 12:37:18 2023
************************************************************************/
package main

import (
        "fmt"
        "runtime"
)

func main() {
        fmt.Printf("Hello, %s!\\n", runtime.GOARCH)
}
```

创建一个 Dockerfile 将该应用容器化：

```yaml
❯ cat Dockerfile
FROM golang:alpine AS builder
RUN mkdir /app
ADD . /app/
WORKDIR /app
RUN go build -o hello .

FROM alpine
RUN mkdir /app
WORKDIR /app
COPY --from=builder /app/hello .
CMD ["./hello"]
```

这是一个多阶段构建 Dockerfile，使用 Go 编译器来构建应用，并将构建好的二进制文件拷贝到 alpine 镜像中。

现在就可以使用 buildx 构建一个支持 arm、arm64 和 amd64 多架构的 Docker 镜像了，同时将其推送到 **Docker Hub**：

```yaml
→ docker buildx build -t cubxxw/hello-arch --platform=linux/arm,linux/arm64,linux/amd64 . --push
```

> 需要提前通过 docker login 命令登录认证 Docker Hub。

现在就可以通过 `docker pull mirailabs/hello-arch` 拉取刚刚创建的镜像了，Docker 将会根据你的 CPU 架构拉取匹配的镜像。

背后的原理也很简单，之前已经提到过了，buildx 会通过 `QEMU` 和 `binfmt_misc` 分别为 3 个不同的 CPU 架构（arm，arm64 和 amd64）构建 3 个不同的镜像。构建完成后，就会创建一个 **manifest** ，其中包含了指向这 3 个镜像的指针。

现在就可以通过 `docker pull mirailabs/hello-arch` 拉取刚刚创建的镜像了，Docker 将会根据你的 CPU 架构拉取匹配的镜像。

背后的原理也很简单，之前已经提到过了，buildx 会通过 `QEMU` 和 `binfmt_misc` 分别为 3 个不同的 CPU 架构（arm，arm64 和 amd64）构建 3 个不同的镜像。构建完成后，就会创建一个 **manifest list**，其中包含了指向这 3 个镜像的指针。

**保存在本地：**

如果想将构建好的镜像保存在本地，可以将 `type` 指定为 `docker`，但必须分别为不同的 CPU 架构构建不同的镜像，不能合并成一个镜像，即：

```yaml
→ docker buildx build -t cubxxw/hello-arch --platform=linux/arm -o type=docker .
→ docker buildx build -t cubxxw/hello-arch --platform=linux/arm64 -o type=docker .
→ docker buildx build -t cubxxw/hello-arch --platform=linux/amd64 -o type=docker .
```

### **测试多平台镜像**

由于之前已经启用了 `binfmt_misc`，现在我们就可以运行任何 CPU 架构的 Docker 镜像了，因此可以在本地系统上测试之前生成的 3 个镜像是否有问题。

首先列出每个镜像的 `digests`：

```yaml
? → docker buildx imagetools inspect cubxxw/hello-arch

Name:      docker.io/cubxxw/hello-arch:latest
MediaType: application/vnd.docker.distribution.manifest.list.v2+json
Digest:    sha256:ec55f5ece9a12db0c6c367acda8fd1214f50ee502902f97b72f7bff268ebc35a

Manifests:
  Name:      docker.io/cubxxw/hello-arch:latest@sha256:38e083870044cfde7f23a2eec91e307ec645282e76fd0356a29b32122b11c639
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/arm/v7

  Name:      docker.io/cubxxw/hello-arch:latest@sha256:de273a2a3ce92a5dc1e6f2d796bb85a81fe1a61f82c4caaf08efed9cf05af66d
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/arm64

  Name:      docker.io/cubxxw/hello-arch:latest@sha256:8b735708d7d30e9cd6eb993449b1047b7229e53fbcebe940217cb36194e9e3a2
  MediaType: application/vnd.docker.distribution.manifest.v2+json
  Platform:  linux/amd64
```

运行每一个镜像并观察输出结果：

```yaml
? → docker run --rm docker.io/cubxxw/hello-arch:latest@sha256:38e083870044cfde7f23a2eec91e307ec645282e76fd0356a29b32122b11c639
Hello, arm!

? → docker run --rm docker.io/cubxxw/hello-arch:latest@sha256:de273a2a3ce92a5dc1e6f2d796bb85a81fe1a61f82c4caaf08efed9cf05af66d
Hello, arm64!

? → docker run --rm docker.io/cubxxw/hello-arch:latest@sha256:8b735708d7d30e9cd6eb993449b1047b7229e53fbcebe940217cb36194e9e3a2
Hello, amd64!
```

## **[buildx 的跨平台构建策略](https://waynerv.com/posts/building-multi-architecture-images-with-docker-buildx/#contents:buildx-的跨平台构建策略)**

根据构建节点和目标程序语言不同，`buildx` 支持以下三种跨平台构建策略：

1. 通过 QEMU 的用户态模式创建轻量级的虚拟机，在虚拟机系统中构建镜像。
2. 在一个 builder 实例中加入多个不同目标平台的节点，通过原生节点构建对应平台镜像。
3. 分阶段构建并且交叉编译到不同的目标架构。

QEMU 通常用于模拟完整的操作系统，它还可以通过用户态模式运行：以 `binfmt_misc` 在宿主机系统中注册一个二进制转换处理程序，并在程序运行时动态翻译二进制文件，根据需要将系统调用从目标 CPU 架构转换为当前系统的 CPU 架构。最终的效果就像在一个虚拟机中运行目标 CPU 架构的二进制文件。Docker Desktop 内置了 QEMU 支持，其他满足运行要求的平台可通过以下方式安装：

```yaml
docker run --privileged --rm tonistiigi/binfmt --install all
```

## OpenIM 跨平台编译实战

我们需要制作 OpenIM 离线部署的方案，首先来说，我们需要熟悉 OpenIM 部署需要哪些组件，查看

| Service Name       | Image                                   | Supported Architectures | Ports                   |
| ------------------ | --------------------------------------- | ----------------------- | ----------------------- |
| mysql              | mysql:5.7                               | amd64, arm64v8, arm32v7 | 13306:3306, 23306:33060 |
| mongodb            | mongo:4.0                               | amd64, arm64v8, arm32v7 | 37017:27017             |
| redis              | redis                                   | amd64, arm64v8, arm32v7 | 16379:6379              |
| zookeeper          | wurstmeister/zookeeper                  | amd64                   | 2181:2181               |
| kafka              | wurstmeister/kafka                      | amd64, arm              | 9092:9092               |
| etcd               | http://quay.io/coreos/etcd              | amd64, arm64v8          | 2379:2379, 2380:2380    |
| minio              | minio/minio                             | amd64, arm64v8, arm32v7 | 10005:9000, 9090:9090   |
| open_im_server     | openim/open_im_server:v2.3.9            | amd64                   | N/A                     |
| open_im_enterprise | openim/open_im_enterprise:v1.0.3        | amd64                   | N/A                     |
| prometheus         | prom/prometheus                         | amd64, arm64v8, arm32v7 | N/A                     |
| grafana            | grafana/grafana                         | amd64, arm64v8, arm32v7 | N/A                     |
| node-exporter      | http://quay.io/prometheus/node-exporter | amd64, arm64v8, arm32v7 | 9100:9100               |

注意看， zookeeper 和 openim 并没有提供 arm 架构的设计方案。

所以我们需要自己去编译 arm 架构的镜像，这一层设计比较复杂。为了形成构建的自动化，我们将使用 CICD 和 Makefile 集成。
