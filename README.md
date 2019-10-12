# 采用 Kubebuilder 和 client-go 编写 K8S CRD 教程

## 第一步，创建仓库

1. 创建空目录
2. go mod init tutorial.kubebuilder.io/project

我们将会得到一个空的 Go 项目仓库，模块导入路径已被设置为 tutorial.kubebuilder.io/project

## 第二步，使用 Kubebuilder 创建项目模版

1. kubebuilder init --domain tutorial.kubebuilder.io

将会得到一个项目模版，其中包含了:

* `.gitignore` 文件，该文件忽略了那些常见的 K8S 项目中的临时文件，让你的 Git 仓库更干净，更有利于跟踪源代码的修改。
* 更新后的 `go.mod` 文件，包含了新的模块依赖
* `PROJECT` 文件，这里面是一些 Kubebuilder 元信息，用来搭建新组件
* `Makefile` 文件，用来编译和部署你的控制器
* `Dockerfile` 文件，控制器镜像的生成脚本，配合 `Makefile` 使用。
* 启动配置，`config/*` 目录下有一堆的启动配置，用作不同的使用场景
    - `config/default/*` 用来在标准配置下启动控制器
    - `config/manager/*` 用来在集群中以 Pod 的方式启动控制器
    - `config/rbac/*` 如果控制器有自己的账户服务，需要权限认证的话，用这一组配置
* 当然，最重要的还是项目入口文件 `main.go`

### 项目入口

在 `main.go` 中，主要做了以下四件事：

1. 创建 Scheme
2. 为监控设置命令行参数
3. 创建 Manager
4. 启动 Manager
    - 管理器会启动控制器和 Webhook

注意在 `main.go` 中有一些 `// +kubebuilder:....` 格式的注释，这些注释是一些特殊的占位符，
将会在编译环节自动生成更多的代码。

## 等等，先讲一些概念

当我们谈论 K8S 中的 API 时，有四个重要的概念需要先了解一下：

* **Group**，就是一组 API 的集合
* **Version**，每个 Group 都有若干个 Version，随着时间的推移，Version 会越来越多。
* **Kind**，每一组 API 的每个版本都包含了一个或者多个数据类型，称之为 Kind。随着版本的变化，Kind 可能会产生不同的变种，
  但任何一个变种都应当能够包含所有其它变种的所有数据，这样即使是旧的 API，也不会遗失或者破坏新的数据。
* **Resource**，Resource 只是 API 中对 Kind 的使用。通常 Kind 和 Resource 总是一一对应，命名上 Kind 习惯首字母大写，而 Resource 是对应的全小写版本。
  不过也有例外，有时候多个不同的 Resource 可能会返回相同的 Kind。

在 Go 语言中，Group-Version-Kind 的组合经常称之为 **GVK**，而 Group-Version-Resouce 的组合则称之为 **GVR**。

## 第三步，添加一个 API

通过运行下面的命令，可以添加一个新的 Kind：

```
kubebuilder create api --group batch --version v1 --kind CronJob
```

运行完上面的命令之后，会产生如下变化：

```
 M        PROJECT
 A        api/v1/cronjob_types.go
          api/v1/zz_generated.deepcopy.go
 A        api/v1/groupversion_info.go
 A        config/crd/kustomization.yaml
 A        config/crd/kustomizeconfig.yaml
 A        config/crd/patches/cainjection_in_cronjobs.yaml
 A        config/crd/patches/webhook_in_cronjobs.yaml
 A        config/samples/batch_v1_cronjob.yaml
 A        controllers/cronjob_controller.go
 A        controllers/suite_test.go
 M        go.mod
 M        main.go
```

主要有三类变化：

* api/v1 下有三个文件，其中 cronjob_types.go 需要我们自己填充内容，另外两个不用管。
* 生成了一个 controller 框架，这个也需要我们完善。
* yaml 暂时也不用管。

### CronJob Kind

Kind 都在 api/v1/cronjob_types.go 文件中，其中有四个数据类型：

* CronJob 代表单数的 CronJob 对象，也就是我们将要实现的计划任务
* CronJobList 代表复数的 CronJob 对象，里面包含多个 CronJob
* CronJobSpec 代表计划任务的预期状态
* CronJobStatus 代表计划任务的实际状态

在 K8S 中，控制器主要是通过比对预期状态和实际状态的差别，来形成指令。

### CronJob 控制器

每个控制器都负责维护一个对象，通过比较预期状态和实际状态，来触发具体的动作，
这个过程称之为「协调」，相对应的数据类型叫做「协调器」。

kubebuilder 为我们生成了协调器的框架代码，主要包括：

* 数据类型定义
* 两个主要的方法： Reconcile 和 SetupWithManager

接下来我们就去完善它们。
