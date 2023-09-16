---
title : '速读开源项目 Sealos 的源码'
description:
ShowRssButtonInSectionTermList: true
cover.image:
date : 2023-09-16T16:33:09+08:00
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



## 准备

这篇文章等的太久了，大致 四个月了把，也是自己经历 docker 跨越到 Kubernetes 以及 CloudNative 生态的过程。

反过来再去理解开源、理解 sealos、 理解 Kubernetes，有种豁然开朗的视角。

这篇文章和其他文章不一样的是，这篇是按照自己现在的思路来写的，具体为什么，在以前的文章中能找到答案~

**从 CMD 角度上对接源码，从最开始出发：**

不管是 sealer 还是 sealctl，都离不开 镜像的构建核心》 buildah：

```go
package main

import (
	"github.com/containers/buildah"

	"github.com/labring/sealos/cmd/sealctl/cmd"
)

func main() {
	if buildah.InitReexec() {
		return
	}
	cmd.Execute()
}
```

从 `InitReexec` 调用 buildah 初始化开始，进行走进 sealos 的大门：`Execute`

在 cobra 中，`Execute` 只会执行一次，不管是正确的还是失败的~

在调用的时候，会先执行  `init` 初始化函数，它 定义了一些初始化工作以及标志：

```go
func init() {
	cobra.OnInitialize(func() {
		logger.CfgConsoleLogger(debug, showPath)
	})

	rootCmd.PersistentFlags().BoolVar(&debug, "debug", false, "enable debug logger")
	rootCmd.PersistentFlags().BoolVar(&showPath, "show-path", false, "enable show code path")
}
```

哈哈，sealos 对于日志包的封装，还是很让我惊喜的，使用了 `zap` 进行二次开发和封装，用于适合自己的业务需要，这对我是有参考意义的，包括 horizon，未来可能需要在 日志包和 错误码设计上进行改进，这是成就一个优秀的开源项目的必要条件~

后面对 `rootCmd` 进行了标志绑定，这就是我们 `sealos` 的根命令：

```go
// rootCmd represents the base command when called without any subcommands
var rootCmd = &cobra.Command{
	Use:   "sealos",
	Short: "sealos is a Kubernetes distribution, a unified OS to manage cloud native applications.",
}
```

作为 sealos 成就 Kubernetes 集群最主要的命令，我们来尝试一下 `sealos run`

+ [使用 sealos 快速搭建 HA cluster](https://docker.nsddd.top/Cloud-Native-k8s/6.html)

```bash
sealos run labring/kubernetes:v1.25.0 labring/helm:v3.8.2 labring/calico:v3.24.1 \
     --masters 192.168.0.2,192.168.0.3\
     --nodes 192.168.0.4 -p [your-ssh-passwd]
```

很高兴的一点，相比较 sealer，sealos 将 run.go 逻辑中的大部分都放在了 pkg 中实现，让其看上去并不是那么臃肿，但是对于 sealos 的架构来说，因为没有 cluster-runtime 作为抽象层，所以 sealos 的依赖过于严重，这也是我目前正在设计 k3s runtime 所必须解决的思路。



## 原理实现

run 简简单单的一个命名是高度抽象的：我们 run 的时候 cmd 将 `applier, err := apply.NewApplierFromArgs(images, runArgs)` 传递给 `NewApplierFromArgs` 

```go
func NewApplierFromArgs(imageName []string, args *RunArgs) (applydrivers.Interface, error) {
	clusterPath := constants.Clusterfile(args.ClusterName)
	cf := clusterfile.NewClusterFile(clusterPath,
		clusterfile.WithCustomConfigFiles(args.CustomConfigFiles),
		clusterfile.WithCustomEnvs(args.CustomEnv),
	)
	err := cf.Process()
	if err != nil && err != clusterfile.ErrClusterFileNotExists {
		return nil, err
	}
	if err = cf.SetSingleMode(args.Single); err != nil {
		return nil, err
	}

	cluster := cf.GetCluster()
	if cluster == nil {
		logger.Debug("creating new cluster")
		cluster = initCluster(args.ClusterName)
	} else {
		cluster = cluster.DeepCopy()
	}
	c := &ClusterArgs{
		clusterName: cluster.Name,
		cluster:     cluster,
	}
	if err = c.runArgs(imageName, args); err != nil {
		return nil, err
	}
	return applydrivers.NewDefaultApplier(c.cluster, cf, imageName)
} 
```

`NewApplierFromArgs` 用于创建一个 `Applier` 实例对象。它接收两个参数：一个是镜像名称数组 `imageName`，另一个是运行参数 args。在这个函数中，它实现了从参数中获取 `ClusterName`，然后根据 `ClusterName` 获取对应的 `ClusterFile`，如果获取不到，则创建一个新的。接着，它根据用户输入的参数，更新集群状态 Cluster 中的 spec，最后通过 Cluster 对象和 ClusterFile 对象，创建一个 Applier 对象返回。



## Applier

首先，Sealos 会创建一个 Applier 结构体，负责了部署集群的核心逻辑。

```go
func NewDefaultApplier(cluster *v2.Cluster, cf clusterfile.Interface, images []string) (Interface, error) {
	if cluster.Name == "" {
		return nil, fmt.Errorf("cluster name cannot be empty")
	}
	if cf == nil {
		cf = clusterfile.NewClusterFile(constants.Clusterfile(cluster.Name))
	}
	err := cf.Process()
	if !cluster.CreationTimestamp.IsZero() && err != nil {
		return nil, err
	}

	return &Applier{
		ClusterDesired: cluster,
		ClusterFile:    cf,
		ClusterCurrent: cf.GetCluster(),
		RunNewImages:   images,
	}, nil
}
```

`NewDefaultApplier` 函数是用于创建一个 `Applier` 实例对象的。它接收三个参数：

+ 一个是 `Cluster` 对象
+ 一个是 `ClusterFile` 对象
+ 还有一个是镜像名称数组 `images`

在这个函数中，它实现了从参数中获取集群名称，然后根据集群名称获取对应的 `ClusterFile`，如果不存在，则返回一个错误。接着，它根据用户输入的参数，更新集群状态 `ClusterDesired` 中的 `Spec`，最后通过 `ClusterDesired` 对象和 `ClusterFile` 对象，创建一个 `Applier` 对象返回。

**具体步骤如下：**

1. 判断集群名称是否为空，如果为空则返回一个错误。
2. 判断 `ClusterFile` 是否为空，如果为空则创建一个新的 `ClusterFile`。
3. 如果 `ClusterDesired` 的创建时间戳为空且 `ClusterCurrent` 为 `nil` 或 `ClusterCurrent` 的创建时间戳为空，则初始化创建一个新的集群，`ClusterDesired` 的创建时间戳设置为当前时间。
4. 如果 `ClusterDesired` 和 `ClusterCurrent` 的创建时间戳都不为空，则更新集群状态 `ClusterDesired` 中的 `Spec`。

`Applier` 采用了 k8s 的 **声明式** 的设计思想，用户声明一个期望的集群状态，而 Applier 负责将集群现在的状态转换成用户期望的状态。



## Applier struct

```go
type Applier struct {
    ClusterDesired     *v2.Cluster // 用户期望的集群状态
    ClusterCurrent     *v2.Cluster // 集群当前状态
    ClusterFile        clusterfile.Interface // 当前集群接口
    Client             kubernetes.Client
    CurrentClusterInfo *version.Info
    RunNewImages       []string              // run 命令新增的镜像名称
}
```

`clusterfile.Interface` 是一个接口类型，Sealos 中通过 `ClusterFile` 实现了这一接口。因此，`Applier` 结构体中最重要的就是 `Cluster` 和 `ClusterFile` 这两个类型，它们定义了集群的状态和配置。

1. ClusterDesired：用户期望的集群状态
2. ClusterCurrent：集群当前状态
3. ClusterFile：当前集群接口
4. Client：Kubernetes 客户端
5. CurrentClusterInfo：集群当前信息
6. RunNewImages：run 命令新增的镜像名称



## 深挖集群的 Cluster 结构体

```go
type Cluster struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ClusterSpec   `json:"spec,omitempty"`
    Status ClusterStatus `json:"status,omitempty"`
}
type ClusterSpec struct {
    Image ImageList `json:"image,omitempty"`
    SSH   SSH       `json:"ssh"`
    Hosts []Host    `json:"hosts,omitempty"`
    Env []string `json:"env,omitempty"`
    Command []string `json:"command,omitempty"`
}
type ClusterStatus struct {
    Phase      ClusterPhase       `json:"phase,omitempty"`
    Mounts     []MountImage       `json:"mounts,omitempty"`
    Conditions []ClusterCondition `json:"conditions,omitempty" `
}
```

Cluster 的内容按照 K8s Resource 的格式进行了设计，可以看到是将结构体都拆分出来了，而不是使用的结构体嵌套，这样更加规范、更加整洁。在 ClusterSpec 中，定义了一系列用于部署 K8s 集群的参数，例如，镜像、SSH 参数、节点等等。

而在 ClusterStatus 中，`Phase` 定义了当前集群的状态，`Mounts` 定义了集群使用的镜像，`Conditions` 保存了集群中所发生的一系列事件。



## ClusterFile

`ClusterFile` 是真正被 `Applier` 操作的对象，以及持久化到文件中的内容。这里包含了所有集群的当前状态信息，同时还包含了 `kubeconfig`。这里的 kubeconfig 并不是我们平时操作 k8s 时所用的 config 文件，而是一系列用于搭建集群所需的配置项。在使用 `kubeadm` 时，这些配置项往往需要我们手动配置，而 Sealos 在这里会自动帮我们填写并应用于集群中。可以看出，`Cluster` 更像是 `ClusterFile` 的一个实例，记录了集群实时的状态。

```go
type Interface interface {
	PreProcessor
	GetCluster() *v2.Cluster
	GetConfigs() []v2.Config
	GetKubeadmConfig() *runtime.KubeadmConfig
}
```



## 创建 Applier

创建 Applier 的逻辑：

`buildah mount` 命令是用于将容器镜像挂载到本地文件系统上的工具。通过该命令可以方便地查看、编辑容器镜像中的文件。具体用法可以参考 [官方文档](https://buildah.io/commands/mount/)。

![Untitled](http://sm.nsddd.top/sm202304152156333.png)

**创建一个 `Applier` 会经过以下步骤：**

1. 判断是否已经存在 `ClusterFile` ，如果存在，那么直接读取，构建出集群状态 `Cluster`。否则，初始化创建一个空的集群状态 `Cluster`。
2. 根据用户本次的参数，更新集群状态 `Cluster` 中的 spec，此时，`Cluster` 即为目标的集群状态。
3. 再次从文件中构建 `ClusterFile`，作为集群当前的状态和对象。
4. 构建 `Applier` 结构体返回。

这个时候我们就回到了 run 的逻辑：

```go
RunE: func(cmd *cobra.Command, args []string) error {
			if runSingle {
				addr, _ := iputils.ListLocalHostAddrs()
				runArgs.Masters = iputils.LocalIP(addr)
				runArgs.Single = true
			}

			images, err := args2Images(args, transport)
			if err != nil {
				return err
			}

			applier, err := apply.NewApplierFromArgs(images, runArgs)
			if err != nil {
				return err
			}
			return applier.Apply()
		},
```

`applier` 就是构建出的结构体：构建 `applier` 的目的是为了将集群的当前状态转换成用户期望的状态。通过初始化创建一个 `Applier`，可以根据用户本次的参数，更新集群状态 `Cluster` 中的 spec，最终构建出一个 `Applier` 结构体。这个结构体可以将用户期望的集群状态转换成实际的集群状态，实现了 K8s 的声明式的设计思想。接下来到了 `Apply()` 操作了。



## 开始 Apply

接下来，通过 `Applier.Apply()`，Sealos 开始正式的部署集群，使集群状态向目标靠近。首先，Sealos 会将当前集群的状态置为 `ClusterInProcess`。接下来，根据集群创建或是更新，分别进入两个分支。

> 这里有一点需要说明得，之前有一次面试的时候面试官问我，`apply` 和 `create` 实现的逻辑，这两个实现的逻辑区别就是，很明显的，create 并不会进行 控制器 的 观察、分析阶段，而是直接执行和更新，这样的是符合的是命令式特点，而并非是声明式的 特征。

```go
func (c *Applier) Apply() error {
	clusterPath := constants.Clusterfile(c.ClusterDesired.Name)
	// clusterErr and appErr should not appear in the same time
	var clusterErr, appErr error
	// save cluster to file after apply
	defer func() {
		switch clusterErr.(type) {
		case *processor.CheckError, *processor.PreProcessError:
			return
		}
		logger.Debug("write cluster file to local storage: %s", clusterPath)
		saveErr := yaml.MarshalYamlToFile(clusterPath, c.getWriteBackObjects()...)
		if saveErr != nil {
			logger.Error("write cluster file to local storage: %s error, %s", clusterPath, saveErr)
			logger.Debug("complete write back file: \n %v", c.getWriteBackObjects())
		}
	}()
	c.initStatus()
	if c.ClusterDesired.CreationTimestamp.IsZero() && (c.ClusterCurrent == nil || c.ClusterCurrent.CreationTimestamp.IsZero()) {
		clusterErr = c.initCluster()
		c.ClusterDesired.CreationTimestamp = metav1.Now()
	} else {
		clusterErr, appErr = c.reconcileCluster()
		c.ClusterDesired.CreationTimestamp = c.ClusterCurrent.CreationTimestamp
	}
	c.updateStatus(clusterErr, appErr)

	// return app error if not nil
	if appErr != nil && !errors.Is(appErr, processor.ErrCancelled) {
		return appErr
	}
	return clusterErr
}
```

`Apply()` 函数是 Sealos 中负责部署集群的核心函数。该函数通过传入的 `Applier` 实例的 `ClusterDesired` 和 `ClusterCurrent` 的值来判断集群是否已经存在。在函数执行时，首先会将当前集群状态置为 `ClusterInProcess`，然后分别调用 `initCluster()` 和 `reconcileCluster()` 来 **创建和更新集群**。最后，函数会根据 `appErr` 和 `clusterErr` 的值来更新集群（或者 APP）的状态，因为上面讲过 `EXECUTE` 只会执行一次，无论对错，所以 cluster 和 app 只能有一个存在。



为什么需要 `c.initStatus()`:

initStatus 函数的作用是初始化集群状态，即为集群状态 Cluster 中的 Status 字段赋初值：将 Phase 设为 ClusterInProcess，如果 Conditions 为空，则创建一个空数组。

```go
func (c *Applier) initStatus() {
	c.ClusterDesired.Status.Phase = v2.ClusterInProcess
	if c.ClusterDesired.Status.Conditions == nil {
		c.ClusterDesired.Status.Conditions = make([]v2.ClusterCondition, 0)
	}
}
```

函数实现很简单，先将 `Phase` 设为 `ClusterInProcess`，表示集群正在部署中。然后，判断 `Conditions` 是否为空，如果为空，则将 `Conditions` 设为一个空数组。

这段代码是在判断集群是否已经存在，具体解释如下：

+ `c.ClusterDesired.CreationTimestamp.IsZero()` 判断期望状态的 `CreationTimestamp` 是否为空，如果为空则说明集群不存在或者还没有创建。
+ `(c.ClusterCurrent == nil || c.ClusterCurrent.CreationTimestamp.IsZero())` 判断当前状态的 `ClusterCurrent` 是否为空或者 `CreationTimestamp` 是否为空，如果为空则说明集群不存在或者还没有创建。

如果以上两个条件都满足，说明集群还没有创建，可以调用 `initCluster()` 函数来创建一个新的集群。否则，说明集群已经存在，可以调用 `reconcileCluster()` 函数来更新集群。

`yaml.MarshalYamlToFile` 的作用是将一个或多个对象序列化为 YAML 格式，并将序列化后的字符串写入指定的文件。在 Sealos 中，`yaml.MarshalYamlToFile` 用于将 `ClusterFile` 对象序列化为 YAML 格式，并将其写入指定的文件中。这个文件用于持久化集群的状态。

`initCluster()` 函数会从零开始创建一个集群，会使用 `CreateProcessor` 对象来部署期望状态的集群。`CreateProcessor.Execute()` 函数接收期望的集群状态 `ClusterDesired`，然后执行一系列 pipeline，正式进入实际的集群部署过程中。

```go
func (c *InstallProcessor) GetPipeLine() ([]func(cluster *v2.Cluster) error, error) {
	var todoList []func(cluster *v2.Cluster) error
	todoList = append(todoList,
		c.SyncStatusAndCheck,
		c.ConfirmOverrideApps,
		c.PreProcess,
		c.RunConfig,
		c.MountRootfs,
		c.MirrorRegistry,
		c.UpgradeIfNeed,
		// i.GetPhasePluginFunc(plugin.PhasePreGuest),
		c.RunGuest,
		c.PostProcess,
		// i.GetPhasePluginFunc(plugin.PhasePostInstall),
	)
	return todoList, nil
}
```

在 Sealos 中，`CreateProcessor` 和 `InstallProcessor` 是两个不同的 Processor，分别用于创建集群和安装集群。它们实现了 `processor.Processor` 接口，该接口定义了执行集群部署所需的基本方法。每个 Processor 中都包含了一系列的 Pipeline，每个 Pipeline 中又包含了一系列的函数，这些函数用于执行具体的部署操作。

在 `CreateProcessor` 中，`GetPipeLine()` 函数返回一个包含了创建集群所需 Pipeline 的列表。这个列表中包含了一些基本操作，例如检查集群是否已经存在、运行配置、检查并挂载 rootfs、启动 bootstrap 等等。这些操作会被依次执行，最终完成集群的创建过程。

而在 `InstallProcessor` 中，`GetPipeLine()` 函数返回一个包含了安装集群所需 Pipeline 的列表。这个列表中包含了与创建集群类似的操作，但也包含了一些额外的操作，如升级、运行 guest、后处理等等。这些操作会被依次执行，最终完成集群的安装过程。

从功能上来看，`CreateProcessor` 更侧重于创建集群，而 `InstallProcessor` 更侧重于安装集群。实际上，在 Sealos 中，这两个 Processor 之间并没有太大的区别，它们都包含了相同的 Pipeline 和操作。唯一的区别在于它们被用于不同的场景。

`pipeline` 主要分为以下几个步骤：

1. Check：检查集群的主机，包括 IP 是否能够访问，主机是否为 Linux 系统，用户是否为 root 等。

2. PreProcess：负责集群部署前的镜像预处理操作，主要是使用 `image.Manager` 对象来处理镜像。在这里会拉取镜像，并对镜像进行格式检查、挂载到 rootfs 上等操作。

3. RunConfig：将集群状态中的 `working container` 导出成 yaml 格式的配置并持久化到宿主机的文件系统中。

   ```go
   func (c *InstallProcessor) RunConfig(_ *v2.Cluster) error {
   	if len(c.NewMounts) == 0 {
   		return nil
   	}
   	eg, _ := errgroup.WithContext(context.Background())
   	for _, cManifest := range c.NewMounts {
   		manifest := cManifest
   		eg.Go(func() error {
   			cfg := config.NewConfiguration(manifest.ImageName, manifest.MountPoint, c.ClusterFile.GetConfigs())
   			return cfg.Dump()
   		})
   	}
   	return eg.Wait()
   }
   ```

   `RunConfig` 函数的作用是将集群状态中的 `working container` 导出成 yaml 格式的配置并持久化到宿主机的文件系统中。在函数执行时，会将 `working container` 的配置导出成 Config 对象，然后使用 `config.Dump()` 函数将 Config 对象序列化为 YAML 格式，并将其写入指定的文件中。在 Sealos 中，`RunConfig` 函数主要用于生成 Kubernetes 集群的配置文件，这些配置文件用于持久化集群的状态。

   `rootfs → Clusterfile`

   ```go
   func (c *Dumper) Dump() error {
   	if len(c.Configs) == 0 {
   		logger.Debug("clusterfile config is empty!")
   		return nil
   	}
   
   	if err := c.WriteFiles(); err != nil {
   		return fmt.Errorf("failed to write config files %v", err)
   	}
   	return nil
   }
   
   func (c *Dumper) WriteFiles() (err error) {
   	for _, config := range c.Configs {
   		if config.Spec.Match != "" && config.Spec.Match != c.name {
   			continue
   		}
   		configData := []byte(config.Spec.Data)
   		configPath := filepath.Join(c.RootPath, config.Spec.Path)
   		//only the YAML format is supported
   		switch config.Spec.Strategy {
   		case v1beta1.Merge:
   			configData, err = getMergeConfigData(configPath, configData)
   			if err != nil {
   				return err
   			}
   		case v1beta1.Insert:
   			configData, err = getAppendOrInsertConfigData(configPath, configData, true)
   			if err != nil {
   				return err
   			}
   		case v1beta1.Append:
   			configData, err = getAppendOrInsertConfigData(configPath, configData, false)
   			if err != nil {
   				return err
   			}
   		case v1beta1.Override:
   		}
   		err = file.WriteFile(configPath, configData)
   		if err != nil {
   			return fmt.Errorf("write config file failed %v", err)
   		}
   	}
   
   	return nil
   }
   ```

   `Dump()` 函数是一个将集群配置写入文件的方法。该函数首先检查配置是否为空。如果不为空，它将调用 `WriteFiles()` 方法，该方法将为每个配置文件写入文件。如果没有发生错误，则返回 `nil`。如果出现错误，则返回错误。

   `WriteFiles()` 方法是将集群配置写入文件的实际方法。该方法遍历所有配置并检查与当前名称匹配的配置。如果找到符合条件的配置，则使用该配置中的数据将其写入文件。在写入到文件之前，还需要检查配置中的策略并进行相应的处理。最后，该方法将返回 `nil` 或错误。

   ```go
   const (
   	Merge    StrategyType = "merge"
   	Override StrategyType = "override"
   	Insert   StrategyType = "insert"
   	Append   StrategyType = "append"
   )
   ```

   这些常量定义了 `StrategyType` 类型的枚举值，用于指定在更新配置文件时使用的策略。其中，`Merge` 表示将新配置合并到旧配置中，`Override` 表示使用新配置覆盖旧配置，`Insert` 表示在旧配置文件的指定位置插入新配置，`Append` 表示将新配置追加到旧配置文件的末尾。

4. `MountRootfs`：将挂载的镜像内容按照类别分发到每台机器上。这里需要介绍一下 sealos 的镜像结构，以最基础的 k8s 镜像为例：

```
labring/kubernetes
   - etc # 配置项
   - scripts # 脚本
       - init-containerd.sh
       - init-kube.sh
       - init-shim.sh
       - init-registry.sh
       - init.sh
   - Kubefile # dockerfile 语法，定义了镜像的执行逻辑
```

> 这个过程对我们制作 k3s 的镜像来说非常重要，我在下一个标题详细解释这个步骤，或许我们应该好好的理解这个步骤~



## mount

上面有一部分是 mount 的基础镜像，看到了 `init.sh` 是最开始执行的。这个时候会用 `init` 初始化一些 kubeadm 和 kubectl 一些东西，我们在学习 Kubernetes 中也知道了，Kubernetes 的一些初始化问题，是没有办法使用容器化部署的，因为 kubeadm 是和 容器和 宿主机打交道的。

Sealos 的镜像结构中包含了 addons 文件夹，其中存放了一些额外的组件，比如 dashboard 和 metrics-server。在 MountRootfs 这步中，会执行 addons 类型的 init.sh 脚本，将 addons 中的组件安装到集群中

K8s 作为整个集群的基础，虽然最终镜像内的目录结构与其他一致，但其构建过程稍微有所不同。在 CI 中，我们可以看到，k8s 镜像其实是合并了多个文件夹，containerd，rootfs 和 registry。这些独立的文件夹中包含有安装对应组件的脚本。在 MountRootfs 这步中，只会执行 rootfs 和 addons 类型的 init.sh 脚本。这也很好理解，因为到目前为止，Sealos 仅仅在每台机器上安装成功了 kubelet，整个 k8s 集群还未可用。

先通过一个 构造函数 ：

```go
func (f *defaultRootfs) MountRootfs(cluster *v2.Cluster, hosts []string) error {
	return f.mountRootfs(cluster, hosts)
}
```

虽然我也不知道这个有啥用，不过也算是实现了结构体方法把：

```go
func (f *defaultRootfs) mountRootfs(cluster *v2.Cluster, ipList []string) error {
	target := constants.NewData(f.getClusterName(cluster)).RootFSPath()
	ctx := context.Background()
	eg, _ := errgroup.WithContext(ctx)
	envProcessor := env.NewEnvProcessor(cluster, f.mounts)
	for _, mount := range f.mounts {
		src := mount
		eg.Go(func() error {
			if !file.IsExist(src.MountPoint) {
				logger.Debug("Image %s not exist, render env continue", src.ImageName)
				return nil
			}
			// TODO: if we are planing to support rendering templates for each host,
			// then move this rendering process before ssh.CopyDir and do it one by one.
			err := renderTemplatesWithEnv(src.MountPoint, ipList, envProcessor)
			if err != nil {
				return fmt.Errorf("failed to render env: %w", err)
			}
			dirs, err := file.StatDir(src.MountPoint, true)
			if err != nil {
				return fmt.Errorf("failed to stat files: %w", err)
			}
			if len(dirs) != 0 {
				_, err = exec.RunBashCmd(fmt.Sprintf(constants.DefaultChmodBash, src.MountPoint))
				if err != nil {
					return fmt.Errorf("run chmod to rootfs failed: %w", err)
				}
			}
			return nil
		})
	}
	if err := eg.Wait(); err != nil {
		return err
	}

	sshClient := f.getSSH(cluster)
	notRegistryDirFilter := func(entry fs.DirEntry) bool { return !constants.IsRegistryDir(entry) }

	for idx := range ipList {
		ip := ipList[idx]
		eg.Go(func() error {
			egg, _ := errgroup.WithContext(ctx)
			for idj := range f.mounts {
				mount := f.mounts[idj]
				egg.Go(func() error {
					switch mount.Type {
					case v2.RootfsImage, v2.PatchImage:
						logger.Debug("send mount image, ip: %s, image name: %s, image type: %s", ip, mount.ImageName, mount.Type)
						err := ssh.CopyDir(sshClient, ip, mount.MountPoint, target, notRegistryDirFilter)
						if err != nil {
							return fmt.Errorf("failed to copy %s %s: %v", mount.Type, mount.Name, err)
						}
					}
					return nil
				})
			}
			return egg.Wait()
		})
	}
	err := eg.Wait()
	if err != nil {
		return err
	}

	endEg, _ := errgroup.WithContext(ctx)
	master0 := cluster.GetMaster0IPAndPort()
	for idx := range f.mounts {
		mountInfo := f.mounts[idx]
		endEg.Go(func() error {
			if mountInfo.Type == v2.AppImage {
				logger.Debug("send app mount images, ip: %s, image name: %s, image type: %s", master0, mountInfo.ImageName, mountInfo.Type)
				err = ssh.CopyDir(sshClient, master0, mountInfo.MountPoint, constants.GetAppWorkDir(cluster.Name, mountInfo.Name), notRegistryDirFilter)
				if err != nil {
					return fmt.Errorf("failed to copy %s %s: %v", mountInfo.Type, mountInfo.Name, err)
				}
			}
			return nil
		})
	}
	return endEg.Wait()
}
```

`mountRootfs` 函数是 Sealos 中的一个核心函数，它主要用于将挂载的镜像内容按照类别分发到每台机器上。该函数中的步骤分为以下几个部分：

1. **环境变量处理**：该函数会根据传入的集群和挂载点信息处理环境变量。具体来说，它会创建一个 `envProcessor` 对象，该对象包含了集群的信息以及挂载点的信息，用于处理环境变量。
2. 遍历挂载点：该函数会遍历所有的挂载点，对每个挂载点执行以下操作：
   1. 判断挂载点是否存在。如果不存在，则直接跳过该挂载点。
   2. 渲染环境变量。该函数会将环境变量注入到挂载点中。具体来说，它会调用 `renderTemplatesWithEnv` 函数，该函数会将环境变量注入到挂载点中的模板文件中。
   3. 修改目录权限。如果挂载点中包含子目录，则会将其权限更改为默认权限。具体来说，它会调用 `exec.RunBashCmd` 函数，该函数会执行 `chmod` 命令，将挂载点中的所有目录的权限更改为默认权限。
3. **复制镜像到每个节点**：该函数会将镜像文件夹复制到每个节点上。具体来说，它会遍历每个节点，对于每个节点，它会遍历每个挂载点，将挂载点中的镜像文件夹复制到节点上。具体来说，它会调用 `ssh.CopyDir` 函数，该函数会通过 SSH 将挂载点中的镜像文件夹复制到节点上。
4. **复制应用程序镜像到主节点**：如果存在应用程序镜像，则将其复制到主节点上。具体来说，它会遍历每个挂载点，如果挂载点的类型是应用程序镜像类型，则将其复制到主节点上。具体来说，它会调用 `ssh.CopyDir` 函数，该函数会通过 SSH 将应用程序镜像文件夹复制到主节点上。



## MirrorRegistry 和 Bootstrap 步骤

### MirrorRegistry 

当然，在到 init 的步骤中还有两个阶段：

```
c.MirrorRegistry,
c.Bootstrap,
```

`MirrorRegistry` 阶段是 Sealos 部署集群的一个重要步骤。在这个步骤中，Sealos 会将所需的 Docker 镜像从公共镜像仓库中拉取到本地，并将其推送到本地的 Docker 镜像仓库中。这个步骤的作用是将 Docker 镜像缓存到本地，以避免在集群部署过程中频繁地下载镜像，从而加快集群部署的速度。`MirrorRegistry` 阶段通常是在 `MountRootfs` 阶段之后执行的。

**这个步骤就相当于是 sealos 的核心逻辑，镜像处理的核心逻辑，算是一种黑科技把，就是把 remote docker registery 中的 images pull 到 localhost，然后缓存，后面就可以用缓存的文件了。**



### Bootstrap

`Bootstrap` 阶段是 Sealos 部署集群的最后一个阶段。在这个阶段中，Sealos 会启动 Kubernetes 的初始化程序，对集群进行初始化操作。在初始化过程中，Sealos 会使用 kubeadm 工具来创建 Kubernetes 集群的控制平面和节点，启动各种 Kubernetes 组件，并将它们配置为正常运行。当初始化程序成功完成后，Kubernetes 集群就可以正常使用了。`Bootstrap` 阶段通常是在 `MountRootfs` 和 `MirrorRegistry` 阶段之后执行的。

```go
func (c *CreateProcessor) Bootstrap(cluster *v2.Cluster) error {
	logger.Info("Executing pipeline Bootstrap in CreateProcessor")
	hosts := append(cluster.GetMasterIPAndPortList(), cluster.GetNodeIPAndPortList()...)
	bs := bootstrap.New(cluster)
	return bs.Apply(hosts...)
}
```

该函数的主要作用是创建 bootstrap 实例，并调用其 Apply 函数。其中，cluster 表示 Kubernetes 集群的配置信息，hosts 包含了集群的所有节点的 IP 地址和端口号。

接下来看一下 Sealos 中的 `bootstrap.New` 函数，其代码如下：

```go
func New(cluster *v2.Cluster) Interface {
	ctx := NewContextFrom(cluster)
	bs := &realBootstrap{
		ctx:          ctx,
		preflights:   make([]Applier, 0),
		initializers: make([]Applier, 0),
		postflights:  make([]Applier, 0),
	}
	// register builtin appliers
	_ = bs.RegisterApplier(Preflight, defaultPreflights...)
	_ = bs.RegisterApplier(Init, defaultInitializers...)
	_ = bs.RegisterApplier(Postflight, defaultPostflights...)
	return bs
}
```

该函数的主要作用是创建 bootstrap 实例，并初始化其各个字段的值。其中，cluster 表示 Kubernetes 集群的配置信息，ctx 表示上下文信息，`preflights`、`initializers`、`postflights` 分别表示预检查、初始化、后处理函数的列表。此外，该函数还会调用 `RegisterApplier` 函数，注册默认的预检查、初始化、后处理函数。



### RegisterApplier 函数

接下来看一下 Sealos 中的 RegisterApplier 函数，其代码如下：

```go
func (bs *realBootstrap) RegisterApplier(phase Phase, appliers ...Applier) error {
	switch phase {
	case Preflight:
		bs.preflights = append(bs.preflights, appliers...)
	case Init:
		bs.initializers = append(bs.initializers, appliers...)
	case Postflight:
		bs.postflights = append(bs.postflights, appliers...)
	default:
		return fmt.Errorf("unknown phase %s", phase)
	}
	return nil
}
```

该函数的主要作用是注册预检查、初始化、后处理函数。其中，phase 表示函数的类型，appliers 表示函数列表。该函数会根据 phase 的不同，将函数列表添加到相应的列表中。



### Apply 函数

最后看一下 Sealos 中的 Apply 函数，其代码如下：

```go
func (bs *realBootstrap) Apply(hosts ...string) error {
	appliers := make([]Applier, 0)
	appliers = append(appliers, bs.preflights...)
	appliers = append(appliers, bs.initializers...)
	appliers = append(appliers, bs.postflights...)
	logger.Debug("apply %+v on hosts %+v", appliers, hosts)
	for i := range appliers {
		applier := appliers[i]
		if err := runParallel(hosts, func(host string) error {
			if !applier.Filter(bs.ctx, host) {
				return nil
			}
			logger.Debug("apply %s on host %s", applier, host)
			return applier.Apply(bs.ctx, host)
		}); err != nil {
			return err
		}
	}
	return nil
}
```

该函数的主要作用是依次执行预检查、初始化、后处理函数。其中，`hosts` 表示要执行函数的节点列表，`appliers` 表示要执行的函数列表。该函数会将预检查、初始化、后处理函数分别添加到 `appliers` 列表中，并按照顺序依次执行。在执行过程中，会调用 `Filter` 函数来判断该函数是否需要在该节点上执行。如果需要执行，则会调用 `Apply` 函数来执行该函数。最后，若执行过程中出现错误，则该函数会返回错误信息。

通过对 `Sealos` 的 `Bootstrap` 阶段代码进行分析，我们了解了其调用流程和各个函数的功能。在该阶段中，`Sealos` 会启动 Kubernetes 的初始化程序，对集群进行初始化操作，使 Kubernetes 集群可以正常使用。同时，该阶段也会执行预检查、初始化、后处理函数，以保证集群的正常运行。

> 这里或许可以参考 Linux 内核启动的一个过程，Linux 中的 bootfs 会启动一个 Kernel Boot Process，引导 kernel 的启动，启动后 boot 就会被销毁，生命周期结束。



## init 阶段

Init：初始化 k8s 集群。在这步中，其实也是执行了一系列的子操作。首先，将集群状态写入集群文件中。

### **initCluster**

`initCluster` 负责从零开始创建一个集群。函数中会通过 `CreateProcessor` 去部署期望状态的集群。

```go
type CreateProcessor struct {
    ClusterFile     clusterfile.Interface // 当前集群对象
    ImageManager    types.ImageService // 处理镜像
    ClusterManager  types.ClusterService // 管理 clusterfile
    RegistryManager types.RegistryService // 管理镜像 registry
    Runtime         runtime.Interface   // kubeadm 对象
    Guest           guest.Interface   // 基于 sealos 的应用对象
}
```

`CreateProcessor.Execute` 接收期望的集群状态 `ClusterDesired`。

```bash
func (c *CreateProcessor) Execute(cluster *v2.Cluster) error {
	pipeLine, err := c.GetPipeLine()
	if err != nil {
		return err
	}
	for _, f := range pipeLine {
		if err = f(cluster); err != nil {
			return err
		}
	}

	return nil
}

func (c *CreateProcessor) GetPipeLine() ([]func(cluster *v2.Cluster) error, error) {
	var todoList []func(cluster *v2.Cluster) error
	todoList = append(todoList,
		// c.GetPhasePluginFunc(plugin.PhaseOriginally),
		c.Check,
		c.PreProcess,
		c.RunConfig,
		c.MountRootfs,
		c.MirrorRegistry,
		c.Bootstrap,
		// c.GetPhasePluginFunc(plugin.PhasePreInit),
		c.Init,
		c.Join,
		// c.GetPhasePluginFunc(plugin.PhasePreGuest),
		c.RunGuest,
		// c.GetPhasePluginFunc(plugin.PhasePostInstall),
	)
	return todoList, nil
}
```

+ 方便理解，这里盗用 sealer 图

  ![sdUntitled](http://sm.nsddd.top/sm202304152338621.png)

接下来会执行一系列 pipeline，正式进入实际的集群部署过程中：

1. Check：检查集群的 host
2. PreProcess：负责集群部署前的镜像预处理操作，在这里就会利用 `CreateProcessor` 中的各个 Manager。
3. 拉取镜像
4. 检查镜像格式
5. 使用 `buildah` 从 OCI 格式的镜像中创建 working container，并将容器挂载到 rootfs 上
6. 将容器的 manifest 添加到集群状态中
7. RunConfig：将集群状态中的 working container 导出成 yaml 格式的配置并持久化到宿主机的文件系统中
8. MountRootfs：将挂载的镜像内容按照类别，以 `rootfs`，`addons`，`app` 的顺序分发到每台机器上。 这里需要介绍一下 sealos 镜像的一般结构，以最基础的 k8s 镜像为例：

K8s 作为整个集群的基础，虽然最终镜像内的目录结构与其他一致，但其构建过程稍微有所不同。在 CI **https://github.com/labring/cluster-image/blob/faca63809e7a3eae512100a1eb8f9b7384973175/.github/scripts/kubernetes.sh#L35** 中，我们可以看到，k8s 镜像其实是合并了 cluster-image 仓库下的多个文件夹，`containerd`，`rootfs` 和 `registry`。这些独立的文件夹中包含有安装对应组件的脚本。 Sealos 在挂载一个镜像后，会首先执行 `init.sh` 脚本。例如，以下是 k8s 镜像的脚本中，分别按顺序执行了 `init-containerd.sh` 安装 containerd，`init-shim.sh` 安装 image-cri-shim 和 `init-kube.sh` 安装 kubelet。

```bash
source common.sh
REGISTRY_DOMAIN=${1:-sealos.hub}
REGISTRY_PORT=${2:-5000}

# Install containerd
chmod a+x init-containerd.sh
bash init-containerd.sh ${REGISTRY_DOMAIN} ${REGISTRY_PORT}

if [ $? != 0 ]; then
   error "====init containerd failed!===="
fi

chmod a+x init-shim.sh
bash init-shim.sh

if [ $? != 0 ]; then
   error "====init image-cri-shim failed!===="
fi

chmod a+x init-kube.sh
bash init-kube.sh

logger "init containerd rootfs success"
```

在 MountRootfs 这步中，只会执行 `rootfs` 和 `addons` 类型的 `init.sh` 脚本。这也很好理解，因为到目前为止，Sealos 仅仅在每台机器上安装成功了 kubelet，整个 k8s 集群还未可用。

1. Init：初始化 k8s 集群。在这步中，其实也是执行了一系列的子操作。
2. Sealos 会从 `ClusterFile` 中加载 `kubeadm` 的配置，然后拷贝到 master0 上。
3. 根据 master0 的 hostname 生成证书以及 k8s 配置文件，例如 `admin.conf`，`controller-manager.conf`，`scheduler.conf`，`kubelet.conf`。
4. Sealos 将这些配置以及 rootfs 中的静态文件（主要是一些 policy 的配置）拷贝到 master0 上。
5. Sealos 通过 link 的方式将 rootfs 中的 registry 链接到宿主机的目录上，然后执行脚本 `init-registry.sh`，启动 registry 守护进程。
6. 最后也是最重要的，初始化 master0。首先，将 registry 的域名，api server 的域名（IP 为 master0 的 IP）添加到 master0 宿主机上。然后，调用 `kubeadm init` 创建 k8s 集群。最后，将生成的管理员 kubeconfig 拷贝到 `.kube/config`。
7. Join：使用 kubeadm 将其余 master 和 node 加入现有的集群，然后更新 `ClusterFile`。此时，整个 k8s 集群就已经搭建完毕了。
8. RunGuest: 运行所有类型为 `app` 的镜像的 CMD，安装所有应用。

至此一个 k8s 集群以及基于这个集群的所有应用都被安装完毕。



## controller

+ [https://github.com/labring/sealos/tree/main/controllers](https://github.com/labring/sealos/tree/main/controllers)

如果说 sealos 最核心的部分是什么，我觉得是 controller 的实现，[controllers](https://github.com/labring/sealos/tree/main/controllers) 控制器用来管理集群（k8s 有一些内置的功能 `pod`，deloyment这些，同样可以controllers扩展）：

controllers 使用 Go语言 1.8 +  的新特性：工作区

> 这些功能都是 `k8s` 没有的功能~

1. 我们跑了很多服务器都是通过`infra`管理他们

2. `metering`是用作计量，我们用过多少资源

3. `terminal`就是桌面上的终端应用
4. `user`就是用户的管理，因为`cloud.sealos`是一个多租户的集群
5. `app`: 负责管理用户创建的所有应用，包括创建、删除、更新、查看应用的状态以及部署应用。
6. `cluster`: 负责管理 Kubernetes 集群，包括创建、删除、更新、查看集群的状态以及部署集群。
7. `imagehub`: 负责管理镜像仓库，包括创建、删除、更新、查看镜像仓库的状态以及部署镜像仓库。

我认为这里是  sealos 的核心部分。