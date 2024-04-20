---
title: 'OpenIM离线部署设计'
description: '本文详细介绍了 OpenIM 社区版的离线部署设计方案及其实施策略，旨在帮助读者理解和应用于实际部署过程中。'
ShowRssButtonInSectionTermList: true
date: 2023-05-19T15:20:59+08:00
draft: false
cover:
  image: "http://sm.nsddd.top/sm202309161529285.jpg"
  relative: true
showtoc: true
tocopen: false
type: posts
author: '熊鑫伟，我'
keywords: ['OpenIM', '离线部署', '设计方案', '实施策略', '社区版']
tags: ['OpenIM', '离线部署 (Offline Deployment)', '设计 (Design)', '实施策略 (Implementation Strategy)', '部署 (Deployment)']
categories: ['开发 (Development)']
---

## 1. 基础镜像

以下是你需要的基础镜像及其版本：

- wurstmeister/kafka
- redis:7.0.0
- mongo:6.0.2
- mysql:5.7
- wurstmeister/zookeeper
- minio/minio

使用以下命令拉取这些基础镜像：

```
docker pull wurstmeister/kafka
docker pull redis:7.0.0
docker pull mongo:6.0.2
docker pull mysql:5.7
docker pull wurstmeister/zookeeper
docker pull minio/minio
```

## 2. OpenIM 与 Chat 镜像

**详细了解 OpenIM 和 Chat 的版本管理及存储**: [version.md](https://github.com/OpenIMSDK/Open-IM-Server/blob/main/docs/conversions/version.md)

### OpenIM 镜像

- 获取镜像版本信息: [images.md](https://github.com/OpenIMSDK/Open-IM-Server/blob/main/docs/conversions/images.md)
- 根据所需版本，执行以下命令：

```
docker pull ghcr.io/openimsdk/openim-server:<version-name>
```

### Chat 镜像

- 执行以下命令来拉取镜像：

```
docker pull ghcr.io/openimsdk/openim-server:<version-name>
```

## 3. 镜像存储选择

**存储库**：

- 阿里云：`registry.cn-hangzhou.aliyuncs.com/openimsdk/openim-server`
- Docker Hub：`openim/openim-server`

**版本选择**：

- 稳定版：如 release-v3.2 (或 3.1、3.3)
- 最新版：latest
- main 的最新版：main



## 4. 版本选择

你可以选择以下版本：

- 稳定版：如 release-v3.2
- 最新版：latest
- main 分支的最新版：main

## 5. 离线部署步骤

1. **拉取镜像**: 执行上面的 `docker pull` 命令将所需的所有镜像拉取到本地。
2. **保存镜像**:

```
docker save -o <tar-file-name>.tar <image-name>
```

3. **获取代码**: 克隆仓库：

```
git clone https://github.com/OpenIMSDK/openim-docker.git
```

或从[Releases](https://github.com/OpenIMSDK/openim-docker/releases/)下载代码。

4. **传输文件**: 使用 `scp` 将所有镜像和代码传输到内网服务器。

```
scp <tar-file-name>.tar user@remote-ip:/path/on/remote/server
```

或选择其他传输方式如硬盘。

5. **导入镜像**: 在内网服务器上：

```
docker load -i <tar-file-name>.tar
```

6. **部署**：进入 `openim-docker` 仓库目录，按照 README 文档指导进行部署。

7. **使用 Docker-compose 部署**:

```
docker-compose up -d

# 验证
docker-compose ps
```

> **备注**: 若你使用的是 Docker 的版本 20 之前，需确保已经安装了 `docker-compose`。

## 6. 参考链接

- [OpenIMSDK Issue #432](https://github.com/OpenIMSDK/Open-IM-Server/issues/432)
- [Notion Link](https://nsddd.notion.site/435ee747c0bc44048da9300a2d745ad3?pvs=25)
- [OpenIMSDK Issue #474](https://github.com/OpenIMSDK/Open-IM-Server/issues/474)
