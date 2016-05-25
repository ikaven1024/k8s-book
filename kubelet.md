## 主要调用堆栈

```go
run(s *options.KubeletServer, kcfg *KubeletConfig) (err error)
..UnsecuredKubeletConfig(s *options.KubeletServer) (*KubeletConfig, error)
..CreateAPIServerClientConfig(s *options.KubeletServer) (*restclient.Config, error) // client??
..NewForConfig(c *restclient.Config) (*Clientset, error)  // client
..NewForConfig(c *restclient.Config) (*Clientset, error)  // event client
..RunKubelet(kcfg *KubeletConfig) error
....kcfg.Recorder = eventBroadcaster.NewRecorder(...)     // Event
....capabilities.Setup(kcfg.AllowPrivileged, privilegedSources, 0)  // capabilities
....CreateAndInitKubelet(...)
......makePodSourceConfig(...)      // pod来源：manifest, url, apiserver
......NewMainKubelet(...)
......StartGarbageCollection()
....startKubelet(k KubeletBootstrap, podCfg *config.PodConfig, kc *KubeletConfig)
......(kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate)
........(kl *Kubelet) initializeModules() error  // 
......k.ListenAndServe                 // kubelet server
......k.ListenAndServeReadOnly         // kubelet server(ro)
..http.ListenAndServe()  // Healthz Server
```

## 主要任务

| 任务                | 周期                                   | 说明                                       |
| ----------------- | ------------------------------------ | ---------------------------------------- |
| eventBroadcaster  | <- eventBroadcaster.watch.result     | 打印Event，若EventClient不空则sink到Event        |
| containerGC       | 1min                                 |                                          |
| imageGC           | 5min                                 |                                          |
| logServer         |                                      | 日志服务器：/logs/                             |
| imageManager      | 5min                                 | 镜像使用情况（发现时间，最近使用时间，大小），保存在imageManager.imageRecords结构 |
| containerManager  |                                      | Cgroup。1. ensureState, 1min 2. 更新RuntimeCgroupsName, KubeletCgroupsName, 5min |
| oomWatcher        | <- ow.cadvisor.WatchEvents().channel | 检测OOM Event                              |
| resourceAnalyzer  | 1min                                 | 每个pod启动收集Volume信息的协程                     |
| syncNodeStatus    | 10s                                  | 1. 向APIServer注册node(一次任务)。2. 向APIServer更新node信息 |
| syncNetworkStatus | 30s                                  | CBR0                                     |
| updateRuntimeUp   |                                      |                                          |
| podKiller         |                                      | 从kl.podKillingCh中读取pod，并启动协程杀之           |
| statusManager     | 10s                                  | 1. 更新来自m.podStatusChannel的pod 2. 每10s syncBatch() |
| probeManager      |                                      |                                          |
| pleg              |                                      | pod生存周期事件。                               |
| syncLoop          |                                      |                                          |
| Healthz           | 5s重启                                 | Health server: /healthz                  |

## 任务详析

###syncNodeStatus

1. setNodeAddress: 节点IP地址
2. setNodeStatusInfo
   1. setNodeStatusMachineInfo: 主机信息，包括剩余可分配空间
   2. setNodeStatusVersionInfo: 内核版本、系统或者容器版本，runtime版本，kubelet及kubeproxy版本。
   3. setNodeStatusDaemonEndpoints:  ??
   4. setNodeStatusImages: 镜像列表
3. setNodeOODCondition
4. setNodeReadyCondition
5. recordNodeSchdulableEvent


## Question

**Q1.** Kubelet.podKillingCh 谁往这里扔？

**A1.** Kubelet.syncLoop



**Q1.** m.podStatusChannel 谁往这里扔？

**A1.** Kubelet.syncLoop



## 对象引用

```go
if kubeClient != nil {
    cache.NewReflector(listWatch, &api.Service{}, serviceStore, 0).Run()
    cache.NewReflector(listWatch, &api.Node{}, nodeStore, 0).Run()
}

nodeRef := pkg/kubelet/kubelet.go#ObjectReference{
    Kind:      "Node",
    Name:      nodeName,
    UID:       types.UID(nodeName),
    Namespace: "",
}

containerRefManager := pkg/kubelet/container/container_reference_manager.go#RefManager{}

klet.livenessManager = proberesults.NewManager()
klet.podCache = kubecontainer.NewCache()
klet.podManager = pkg/kubelet/pod/manager.go#basicManager{}
klet.containerRuntime = dockertools.NewDockerManager{}

klet.pleg = pkg/kubelet/pleg/generic.go#GenericPLEG {
    runtime: klet.containerRuntime
    cache: klet.podCache
}

klet.imageManager = pkg/kubelet/image_manager.go#realImageManager {
    runtime:      klet.containerRuntime,
    recorder:     recorder,
    nodeRef:      nodeRef
}

klet.runner = klet.containerRuntime
klet.statusManager = pkg/kubelet/status/manager.go#manager {
    podManager: klet.podManager
}

klet.probeManager = pkg/kubelet/prober/manager.go# manager{
    statusManager:    klet.statusManager,
    prober:           pkg/kubelet/prober/prober.go#prober {
                            runner:     klet.runner,
                            refManager: containerRefManager,
                      }
    readinessManager: pkg/kubelet/prober/results/results_manager.go#manager{},
    livenessManager:  klet.livenessManager
}

klet.podWorkers = pkg/kubelet/pod_workers.go#podWorkers {
    syncPodFn:                 klet.syncPod,
    workQueue:                 klet.workQueue,
    podCache:                  klet.podCache,
}

klet.podKillingCh = make(chan *kubecontainer.PodPair, podKillingChannelCapacity)
```






## 主要源码


```
func startKubelet(k KubeletBootstrap, podCfg *config.PodConfig, kc *KubeletConfig) {
   // start the kubelet
   go wait.Until(func() { k.Run(podCfg.Updates()) }, 0, wait.NeverStop)

   // start the kubelet server
   if kc.EnableServer {
      go wait.Until(func() {
         k.ListenAndServe(kc.Address, kc.Port, kc.TLSOptions, kc.Auth, kc.EnableDebuggingHandlers)
      }, 0, wait.NeverStop)
   }
   if kc.ReadOnlyPort > 0 {
      go wait.Until(func() {
         k.ListenAndServeReadOnly(kc.Address, kc.ReadOnlyPort)
      }, 0, wait.NeverStop)
   }
}
```

KubeletBootstrap:
```
pkg\kubelet\kubelet.go
type Kubelet struct {...}
```

ListenAndServe
```
Starting to listen on %s:%d", address, port
```

ListenAndServeReadOnly
```
"Starting to listen read-only on %s:%d", address, port
REST:
/pods/
/spec/
```