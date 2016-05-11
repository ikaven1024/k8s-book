# kubemark介绍

## 背景

在k8s大集群测试前，环境的准备是个耗时、耗力、耗资的问题：

- 部署麻烦。虽然有salt、fabric等自动化配置管理工具极大地减轻了我们的部署工作量，但是仍然需要一系列的配置，特别是当出现节点故障需要人工恢复。
- 物料需求大。

`kubemark`正是为解决这些问题而生的一个组件，它旨在以仿真的形式在相对小的环境中模拟出大集群环境。

## 原理

k8s node上运行着两个组件：`kubelet`和`kube-proxy`。kube-proxy完成服务发现、负载均衡等工作。kubelet负责与与节点外（如APIServer和heapster等）的交互。在整个集群中kubelet可以看做是node的代理，与node是一一对应的关系。

kubemark（在代码中叫`hollow node`）是一个轻量级的程序，完成了对kubelet和kube-proxy大部分工作的模拟。所谓模拟，它是保留了对外的借口功能，而对于内部操作则实施打桩和`no-op`策略。比如kubelet就”懒“地去收集机子的信息，但对外报告状态时，就输出一份”造假“的报告。这正对应了`hollow(虚伪的)`这词，外强中干！而正是因为这样的功能阉割，才使得kubemark可以很轻量。

在一个机子（物理机或者虚拟机）上启动多个kubemark实例，每个实例都各自向APIServer注册。从外部看来，每个实例被当做一个node。从而达到一台机子在集群中模拟出多个node，缩小了对机子的需求。这对于kube master的功能、性能测试时很有用的，毕竟我的的大部分工作都在master组件上。

## 部署方式

### 1. 进程方式

多个kubemark以进程的形式直接运行在机子上。

![kubemark架构](https://github.com/kubernetes/kubernetes/raw/master/docs/proposals/Kubemark_architecture.png?raw=true)

在这种方式下，有几个问题需要注意：

- 端口冲突
- IP分配
- 实例数维护
- 证书分发

### 2. Pod方式

```
一切皆容器！
```

我们也可以将Kubemark容器化，以上方式的几个问题在k8s机制下都会变得异常简单。但是会受到其他的影响：

- 机子上其他组件（如FluentD, Kubelet）对CPU/Memory的压力影响
- running cluster over an overlay network

### 3. 混合方式

混合前面两种方式来部署，是对两种方式优缺点的折衷。

## 开始部署

### kubemark编译

你也可以在k8s工程下用`make`来编译整个工程，也可以用单独编译Kubemark

```
make all WHAT=cmd/kubemark
```

生成二进制文件在`_output/`下

### 镜像制作

```shell
cd cluster/images/kubemark
cp _output/ .
docker build -t gcr.io/hws/kubemark .
docker save -o gcr.io~hws~kubemark.tar gcr.io/hws/kubemark
```

### 在k8s上部署

获取资源文件，[kubemark-ns.json](https://github.com/kubernetes/kubernetes/blob/master/test/kubemark/resources/kubemark-ns.json)替换[hollow-node-rx.json](https://github.com/kubernetes/kubernetes/blob/master/test/kubemark/resources/hollow-node_template.json)的`##numreplicas##`和`##project##`

```shell
kubectl create -f kubemark-ns.json
kubectl create -f hollow-node-rx.json --namespace=kubemark
```

### 验证

```shell
kubectl get node
```

