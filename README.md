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
