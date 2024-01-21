---
title: 'Kubernetes Kustomize 学习指南'
ShowRssButtonInSectionTermList: true
cover.image:
date: 2023-10-31T21:30:19+08:00
draft: false
showtoc: true
tocopen: true
type: posts
author: ["Xinwei Xiong", "Me"]
keywords: []
tags:
  - 博客
  - Kubernetes
  - Kustomize
categories:
  - 开发
  - 博客
description: >
  探索 Kustomize 的强大功能，这是专为 Kubernetes 设计的开源配置管理工具。
  学习如何使用 Kustomize 以声明式的方式自定义 Kubernetes 对象并管理配置。
  了解与 kubectl 和 Helm 的集成，发现 Kubernetes 配置管理的最佳实践。
---


## 介绍

**关于 kustomize**

+ [github 地址](https://github.com/kubernetes-sigs/kustomize)
+ [Get Started ](https://kubectl.docs.kubernetes.io/zh/installation/)

Kustomize 是一个专为 Kubernetes 设计的开源配置管理工具，它能帮助用户定制 Kubernetes 对象，并以声明式的方式管理这些对象，而无需修改原始的 [YAML 文件1](https://devopscube.com/kustomize-tutorial/)。这意味着你可以保留应用和组件的基本设置，同时通过应用名为“补丁”的声明式 YAML 文档来覆盖默认设置，而不会[更改原始文件](https://www.densify.com/kubernetes-tools/kustomize)。Kustomize 提供了一种声明式的方法，符合 Kubernetes 的哲学，并且能够以一种可重用、快速生成、易于调试和可伸缩的方式定制 Kubernetes 配置。

**Kustomize 的主要功能包括:**

+ **声明式配置**: 允许你以声明式的方式定义和管理 Kubernetes 对象，例如部署、Daemonsets、服务、ConfigMaps 等，为多个环境提供支持，而无需修改原始的 YAML 文件
+ **配置层叠**: 通过利用层叠来保留应用和组件的基本设置，并通过覆盖声明式的 YAML 文档（称为补丁）来选择性地覆盖默认设置
+ **集成与独立使用**: Kustomize 可以作为一个独立的工具使用，或者与 kubectl 结合使用。从 `Kubernetes 1.14` 版本开始，kubectl 也开始支持使用 kustomization 文件来管理 Kubernetes 对象

Kustomize 提供了一种自定义 Kubernetes 资源配置的解决方案，该方案摆脱了模板和 DSL。



## 版本关系 & kubectl 集成

要查找kubectl最新版本中嵌入的kustomize版本，请运行 `kubectl version` ：

```bash
$ kubectl version --short --client
Client Version: v1.26.0
Kustomize Version: v4.5.7
```

kubectl v1.14中添加了v2.0.3的kustomize构建流。kubectl中的kustomize流在kubectl v1.21更新到v4.0.5之前一直冻结在v2.0.3。它将定期更新，这些更新将反映在Kubernetes发布说明中。

| Kubectl version | Kustomize version |
| --------------- | ----------------- |
| < v1.14         | n/a               |
| v1.14-v1.20     | v2.0.3 v2.03      |
| v1.21           | v4.0.5 V4.05      |
| v1.22           | v4.2.0 v4.2 0     |
| v1.23           | v4.4.1 V4.1       |
| v1.24           | v4.5.4            |
| v1.25           | v4.5.7            |
| v1.26           | v4.5.7            |
| v1.27           | v5.0.1            |



## 安装

**从源代码安装kustomize CLI，而不克隆存储库**

For `go version` ≥ `go1.17` 对于 `go version` ≥ `go1.17`

```bash
GOBIN=$(pwd)/ GO111MODULE=on go install sigs.k8s.io/kustomize/kustomize/v5@latest
```

> **Note**
> 除了直接使用 kustomize 命令外，kubernetes 自 v1.14 之后也可以使用 `kubectl kustomize`的方式执行 kustomize



## kubectl 资源

### annotation 注释

更新一个或多个资源上的批注，Kubernetes 注解（annotations）为资源提供了附加元数据。与标签（labels）不同，注解不用于选择和查找资源。注解可以存储大量的数据，比如使用工具、库等为资源提供的长描述、声明检查的时间戳、联系信息或其他信息。

**是什么？**

Annotation（注解）是一种将非标识性元数据附加到对象的方式。客户端工具和库（如 `kubectl` 和 Helm）可以检索这些元数据。



**注解与标签（Labels）的区别**

虽然注解和标签都用于附加元数据，但它们在目的上有所不同：

+ **标签**: 标签是用于选择对象和查找满足某些条件的对象集合。
+ **注解**: 注解主要用于存储辅助数据，以便通过工具和库进行检索。



**使用 `kubectl` 添加和修改注解**

要使用 `kubectl` 为资源添加注解，您可以使用 `annotate` 命令。例如：

```bash
kubectl annotate pods my-pod example.com/some-annotation="some value"
```

这会为名为 `my-pod` 的 Pod 添加一个名为 `example.com/some-annotation` 的注解，并将其值设置为 "some value"。



**更新和删除注解**

使用同样的 `annotate` 命令，您可以修改或删除注解。例如，要更改上面示例中的注解值，只需再次运行相同的命令，并为其提供一个新值。如果要删除注解，可以使用 `-` 符号：

```
kubectl annotate pods my-pod example.com/some-annotation-
```



**查询使用注解的资源**

尽管您不能直接使用 `kubectl` 查询特定的注解值，但您可以使用 `kubectl get` 命令和 `-o json` 或 `-o yaml` 输出格式选项查看资源的所有注解。

```
kubectl get pods my-pod -o=jsonpath='{.metadata.annotations}'
```



## Kustomize 使用

在一些包含YAML资源文件（部署、服务、映射等）的目录中，创建kustomization文件。

当然，Kustomize 和 Helm 可以一起使用，下面是一些使用它们的方法和功能:

1. **HelmChartInflationGenerator**: Kustomize 中内建了一个非常有用的功能叫做 "HelmChartInflationGenerator"，它可以让你在 Kustomize 清单中使用 Helm 图表。当运行 Kustomize 命令时，它会[扩展 Helm 图表以包括 Helm 生成的所有文件](https://medium.com)。
2. **helmCharts 插件**: 你可以直接在 Kustomize 中使用 HelmCharts 插件。例如，你可以将 `values-prod.yaml` 文件放在与 `kustomization.yaml` 文件相同的目录中，然后通过 Kustomize 覆盖 Helm 图表中的默认值。
3. **helm template 和 kubectl kustomize**: 你可以首先使用 `helm template` 命令生成清单，并将其导出到一个文件中，然后运行 `kubectl kustomize` 命令来应用 Kustomize 修改。另一种方式是使用 `helm install` (或 `helm upgrade --install`) 命令，并指定一个自定义的后渲染器来运行 `kubectl kustomize`。
4. **覆盖 Helm 图表**: Kustomize 可以覆盖现有的 Helm 图表，并使用 `HelmChartInflationGenerator` 覆盖一组自定义值。例如，可以使用 Kustomize 部署 Bitnami 的 NGINX Helm 图表，并覆盖默认值以提供自定义的 `nginx.conf` 和自定义的首页。

这个文件应该声明这些资源，以及应用于它们的任何定制，例如。添加一个共同的 lables。

```
base: kustomization + resources
```

File structure: 文件结构：

> ```bash
> ~/someApp
> ├── deployment.yaml
> ├── kustomization.yaml
> └── service.yaml
> ```
>
> 此目录中的资源可能是其他人配置的分支。如果是这样的话，您可以很容易地从源材料中进行改基以获得改进，因为您并不直接修改资源。

生成自定义的YAML：

```bash
kustomize build ~/someApp
```

YAML可以直接应用于集群：

```bash
kustomize build ~/someApp | kubectl apply -f -
```

**和 helm 的区别：**

Kustomize 没有模板语法，只需要一个二进制命令就可以生成对应的 yaml 文件非常的轻量，而 helm 支持 GoTemplate，组件上也要多一些，并且 helm 通过 chart 包来进行发布相对来说还是要重量级一些。个人觉得 Kustomize 更适合做 gitops 而 helm 更合适做应用包的分发。

当然，我们在后面会详细的讨论和 helm 的区别。



### kustomization.yml

一个常见的 `kustomization.yml` 如下所示，一般包含 `apiVsersion` 和 `kind` 两个固定字段

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- manager.yaml

configMapGenerator:
- files:
  - controller_manager_config.yaml
  name: manager-config
```

kustomize 提供了比较丰富的字段选择，除此之外还可以自定义插件，下面会大概列举一下每个字段的含义，当我们需要用到的时候知道有这么个能力，然后再去 [Kustomize 官方文档 ](https://kubectl.docs.kubernetes.io/zh/guides/)查找对应的 API 文档就行了

+ `resources` 表示 k8s 资源的位置，这个可以是一个文件，也可以指向一个文件夹，读取的时候会按照顺序读取，路径可以是相对路径也可以是绝对路径，如果是相对路径那么就是相对于 `kustomization.yml`的路径
+ `crds` 和 `resources` 类似，只是 `crds` 是我们自定义的资源
+ `namespace` 为所有资源添加 namespace
+ `images` 修改镜像的名称、tag 或 image digest ，而无需使用 patches
+ `replicas` 修改资源副本数
+ `namePrefix` 为所有资源和引用的名称添加前缀
+ `nameSuffix` 为所有资源和引用的名称添加后缀
+ `patches` 在资源上添加或覆盖字段，Kustomization 使用 `patches` 字段来提供该功能。
+ `patchesJson6902` 列表中的每个条目都应可以解析为 kubernetes 对象和将应用于该对象的 [JSON patch](https://tools.ietf.org/html/rfc6902)。
+ `patchesStrategicMerge` 使用 strategic merge patch 标准 Patch resources.
+ `vars` 类似指定变量
+ `commonAnnotations` 为所有资源加上 `annotations` 如果对应的 key 已经存在值，这个值将会被覆盖

```yaml
commonAnnotations:
  app.lailin.xyz/inject: agent

resources:
- deploy.yaml
```

`commonLabels` 为所有资源的加上 label 和 label selector **注意：这个操作会比较危险**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

commonLabels:
  app: bingo
```

+ `configMapGenerator` 可以生成 config map，列表中的每一条都会生成一个 configmap
+ `secretGenerator` 用于生成 secret 资源
+ `generatorOptions` 用于控制 `configMapGenerator` 和 `secretGenerator` 的行为



### 注释 Transformer

向所有资源添加注释(annotations)（非标识元数据）。和标签一样，它们也是键值对。

```
commonAnnotations:
  oncallPager: 800-555-1212
```

这个列表中的每个条目都会创建一个ConfigMap资源（它是n个map的生成器）。

下面的示例创建了三个ConfigMap。一个是给定文件的名称和内容，一个是键/值作为数据，第三个是通过 `options` 为单个ConfigMap设置注释和标签。

每个MapGenerator项都接受一个参数 `behavior: [create|replace|merge]` 。这允许覆盖层修改或替换来自父级的现有CNOMAP。

此外，每个条目都有一个 `options` 字段，该字段具有与kustomization文件的 `generatorOptions` 字段相同的子字段。

该 `options` 字段允许向生成的实例添加标签和/或注释，或者单独禁用该实例的名称后缀散列。此处添加的标签和注释不会被与kustomization文件 `generatorOptions` 字段关联的全局选项覆盖。然而，由于布尔值的行为方式，如果全局 `generatorOptions` 字段指定 `disableNameSuffixHash: true` ，这将胜过任何本地覆盖它的尝试。

```yaml
# These labels are added to all configmaps and secrets.
generatorOptions:
  labels:
    fruit: apple

configMapGenerator:
- name: my-java-server-props
  behavior: merge
  files:
  - application.properties
  - more.properties
- name: my-java-server-env-vars
  literals: 
  - JAVA_HOME=/opt/java/jdk
  - JAVA_TOOL_OPTIONS=-agentlib:hprof
  options:
    disableNameSuffixHash: true
    labels:
      pet: dog
- name: dashboards
  files:
  - mydashboard.json
  options:
    annotations:
      dashboard: "1"
    labels:
      app.kubernetes.io/name: "app1"
```

也可以定义一个键来设置不同于文件名的名称。

下面的示例创建了一个ConfigMap，文件名为 `myFileName.ini` ，而创建ConfigMap的实际文件名为 `whatever.ini` 。

```yaml
configMapGenerator:
- name: app-whatever
  files:
  - myFileName.ini=whatever.ini
```



### ImageTagTransformer

图像修改图像的名称、标签和/或摘要，而不创建补丁。例如，给定这个kubernetes Deployment片段：

```yaml
containers:
- name: mypostgresdb
  image: postgres:8
- name: nginxapp
  image: nginx:1.7.9
- name: myapp
  image: my-demo-app:latest
- name: alpine-app
  image: alpine:3.7
```

可以通过以下方式更改 `image` ：

+ `postgres:8` 到 `my-registry/my-postgres:v1` ，
+ nginx标签 `1.7.9` 到 `1.8.0` ，
+ 映像名称 `my-demo-app` 到 `my-app` ，
+ alpine的标签 `3.7` 到摘要值



所有这些都具有以下kustomization：

```yaml
images:
- name: postgres
  newName: my-registry/my-postgres
  newTag: v1
- name: nginx
  newTag: 1.8.0
- name: my-demo-app
  newName: my-app
- name: alpine
  digest: sha256:24a0c4b4a4c0eb97a1aabb8e29f18e917d05abfe1b7a7c07857230879ce7d3d3
```



### *注释Transformer*

向所有资源添加注释（非标识元数据）。和标签一样，它们也是键值对。

```bash
commonAnnotations:
  oncallPager: 800-555-1212
```



### 通过 `transformers` 字段使用

在 Kustomize 中，`transformers` 字段允许您指定一系列转换器，这些转换器可以对原始的资源清单进行修改和调整。

要在 Kustomize 中使用 `transformers`，您需要在 `kustomization.yaml` 文件中指定它，并列出您要使用的转换器配置文件的路径。

例如：

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml

transformers:
- transformers/add-labels.yaml
- transformers/change-image-tag.yaml
```

在上面的示例中，`add-labels.yaml` 和 `change-image-tag.yaml` 将会作为转换器应用，依次修改 `deployment.yaml` 中的资源。

```yaml
apiVersion: builtin
kind: ImageTagTransformer
metadata:
  name: not-important-to-example
imageTag:
  name: nginx
  newTag: v2
```



### LabelTransformer

为所有资源和选择器添加标签

```yaml
commonLabels:
  someName: someValue
  owner: alice
  app: bingo
```



### NamespaceTransformer

将命名空间添加到所有资源

```yaml
namespace: my-namespace
```



## 对比 helm 的使用

Helm 使用的是模板，一个 Helm Chart 包中包含了很多模板和值文件，当被渲染时模板中的变量会使用值文件中对应的值替换。而 Kustomize 使用的是一种无模板的方式，它对 YAML 文件进行修补和合并操作，此外 Kustomize 也已经被原生内置到 kubectl 中了。这两个工具在 Kubernetes 的生态系统中都被广泛使用，**而且这两个工具也可以一起结合使用。**

没错，对于 OpenIM 来说，光光使用 helm 其实也很难满足 OpenIM 的部署需求了，我们更倾向于来使用 Kustomize 。

我们知道很多项目其实都会为应用程序提供 Helm Chart 包，而模板变量的值通过值文件来控制。一个长期存在的问题就是我们应该如何定制上游的 Helm Chart 包，例如从 Helm Chart 包中添加或者一个 Kubernetes 资源清单，如果是通用的变更，最好的选择当然是直接贡献给上游仓库，但是如果是自定义的变更呢？

通常我们可以自己 fork 上游的 Helm Chart 仓库，然后在自己的 repo 中对 Chart 包进行额外的变动。但是这样做，显然会带来额外的负担，特别是当 Chart 包只需要一点小改动的时候。

这个时候我们可以使用 Kustomize 来定制现有的 Helm Chart，而不需要执行 fork 操作。



### Kustomize 插件学习

在Kustomize的GitHub仓库中，有一些插件可以用来扩展其功能。以下是对这些插件的简要介绍：

1. **Exec插件**：此插件可以运行可执行脚本作为一个 [插件](https://github.com/badjware/kustomize-plugins)。
2. **RemoteResources生成器**：此插件可以从远程位置下载 [Kubernetes资源](https://github.com/badjware/kustomize-plugins)。
3. **PlaceholderTransformer转换器**：此插件可以在Kubernetes资源中执行任意的键/值替换。
4. **SSMParameterPlaceholderTransformer转换器**：此插件可以在Kubernetes资源中执行任意的键/值替换，并从AWS系统管理器参数中获取值。
5. **EnvironmentPlaceholderTransformer转换器**：此插件可以在Kubernetes资源中执行任意的键/值替换，并从环境变量中获取值。

其他相关信息包括，用户可以创建转换器或生成器插件，以实现新的行为，这通常意味着需要编写代码，例如Go插件、Go二进制文件、C++ [二进制文件或Bash脚本等](https://github.com/kubernetes-sigs/kustomize/blob/master/examples/configureBuiltinPlugin.md)。在2020年3月时，Kustomize的外部插件还处于alpha功能阶段，所以需要使用`--enable_alpha_plugins`标志来调用构建。

同时，还有一些其他的GitHub仓库也提供了Kustomize插件的集合，例如badjware/kustomize-plugins仓库，sapcc/kustomize-plugins仓库和pollination/kustomize-plugins仓库，其中一些插件可以用来生成Kubernetes secrets，从GCP的密封秘密中生成等。

这些插件通过编写代码，使得用户可以扩展Kustomize的功能，以满足特定的需求，例如通过执行任意的键/值替换来修改Kubernetes资源。



### ChartInflator

Kustomize 提供了一个很好的插件生态系统，允许扩展 Kustomize 的功能。其中就有一个名为 **ChartInflator** 的非内置插件，它允许 Kustomize 来渲染 Helm Charts，并执行任何需要的变更。

**首先先安装 `ChartInflator` 插件：**

```yaml
$ chartinflator_dir="./kustomize/plugin/kustomize.config.k8s.io/v1/chartinflator"

# 创建插件目录
$ mkdir -p ${chartinflator_dir}

# 下载插件
$ curl -L https://raw.githubusercontent.com/kubernetes-sigs/kustomize/kustomize/v3.8.2/plugin/someteam.example.com/v1/chartinflator/ChartInflator > ${chartinflator_dir}/ChartInflator

# 设置插件执行权限
$ chmod u+x ${chartinflator_dir}/ChartInflator
```

比如我们要定制 **Vault Helm Chart** 包，接下来创建 ChartInflator 资源清单和 Helm 的 `values.yaml` 值文件：

```yaml
# ChartInflator 资源清单
$ cat << EOF >> chartinflator-vault.yaml
apiVersion: kustomize.config.k8s.io/v1
kind: ChartInflator
metadata:
  name: vault-official-helm-chart
chartRepo: https://helm.releases.hashicorp.com  
chartName: vault
chartRelease: hashicorp
chartVersion: 0.7.0
releaseName: vault
values: values.yaml
EOF

# 创建 values 值文件
$ helm repo add hashicorp https://helm.releases.hashicorp.com 
$ helm show values --version 0.7.0 hashicorp/vault > values.yaml

# 创建 Kustomize 文件
$ kustomize init
$ cat << EOF >> kustomization.yaml
generators:
- chartinflator-vault.yaml
EOF

# 为所有资源添加一个 label 标签
$ kustomize edit add label env:dev

# 最后生成的 kustomize 文件如下所示：
$ cat kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
generators:
- chartinflator-vault.yaml
commonLabels:
  env: dev

# 整个资源清单目录结构
$ tree .
.
├── chartinflator-vault.yaml
├── kustomization.yaml
├── kustomize
│   └── plugin
│       └── kustomize.config.k8s.io
│           └── v1
│               └── chartinflator
│                   └── ChartInflator
└── values.yaml

5 directories, 4 files
```

现在就可以来渲染 Chart 模板了，执行如下所示的命令即可：

```javascript
$ kustomize build --enable_alpha_plugins .
```

正常渲染完成后我们可以看到所有的资源上都被添加了一个 `env: dev` 的标签，这是实时完成的，不需要维护任何额外的文件的。



### 用单个清单文件定制

另一种使用 Kustomize 定制 Chart 的方法是使用 `helm template` 命令来生成一个单一的资源清单，这种方式可以对 Chart 进行更多的控制，但它需要更多的工作来出来处理更新该生成文件的版本控制。

通常我们可以使用 Make 来进行辅助处理，如下示例所示：

```javascript
# Makefile
CHART_REPO_NAME   := hashicorp
CHART_REPO_URL    := https://helm.releases.hashicorp.com
CHART_NAME        := vault
CHART_VERSION     := 0.7.0
CHART_VALUES_FILE := values.yaml

add-chart-repo:
    helm repo add ${CHART_REPO_NAME} ${CHART_REPO_URL}
    helm repo update

generate-chart-manifest:
    helm template ${CHART_NAME} ${CHART_REPO_NAME}/${CHART_NAME} \
        --version ${CHART_VERSION} \
        --values ${CHART_VALUES_FILE} > ${CHART_NAME}.yaml

get-chart-values:
    @helm show values --version ${CHART_VERSION} \
    ${CHART_REPO_NAME}/${CHART_NAME}

generate-chart-values:
    @echo "Create values file: ${CHART_VALUES_FILE}"
    @$(MAKE) -s get-chart-values > ${CHART_VALUES_FILE}

diff-chart-values:
    @echo "Diff: Local <==> Remote"
    @$(MAKE) -s get-chart-values | \
    diff --suppress-common-lines --side-by-side ${CHART_VALUES_FILE} - || \
    exit 0
```

要定制上游的 Vault Helm Chart，我们可以做如下操作：

```javascript
# 初始化 chart 文件
$ make generate-chart-values generate-chart-manifest 

# 创建 Kustomize 文件并添加一个 label 标签
$ kustomize init
$ kustomize edit add resource vault.yaml
$ kustomize edit add label env:dev

# 最后生成的文件结构如下所示
$ tree .
.
├── kustomization.yaml
├── makefile
├── values.yaml
└── vault.yaml

0 directories, 4 files

# kustomize 文件内容如下所示
$ cat kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- vault.yaml
commonLabels:
  env: dev
```

最后同样用 `kustomize build` 命令来渲染：

```javascript
$ kustomize build .
```

在渲染的结果中同样可以看到所有的资源里面都被添加进了一个 `env: dev` 的标签。

这种方法，需要以某种方式运行 make 命令来生成更新的一体化资源清单文件，另外，要将更新过程与你的 GitOps 工作流整合起来可能有点麻烦。



### 使用 Post Rendering 定制

**Post Rendering** 是 Helm 3 带来的一个新功能，在前面的2种方法中，Kustomize 是用来处理生成图表清单的主要工具，但在这里，Kustomize 是作为 Helm 的辅助工具而存在的。

下面我们来看下如何使用这种方法来进行定制：

```javascript
# 创建 Kustomize 文件并添加一个 label 标签
$ kustomize init
$ kustomize edit add label env:dev

# 创建一个包装 Kustomize 的脚本文件，后面在 Helm 中会使用到
$ cat << EOF > kustomize-wrapper.sh
#!/bin/bash
cat <&0 > chart.yaml
kustomize edit add resource chart.yaml
kustomize build . && rm chart.yaml
EOF
$ chmod +x kustomize-wrapper.sh
```

然后我们可以直接使用 Helm 渲染或者安装 Chart：

```javascript
$ helm repo add hashicorp https://helm.releases.hashicorp.com 
$ helm template vault hashicorp/vault --post-renderer ./kustomize-wrapper.sh
```

正常情况下我们也可以看到最后渲染出来的每一个资源文件中都被添加进了一个 `env:dev` 的标签。

这种方法就是需要管理一个额外的脚本，其余的和第一种方式基本上差不多，只是不使用 Kustomize 的插件，而是直接使用 Helm 本身的功能来渲染上游的 Chart 包。
