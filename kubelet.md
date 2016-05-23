## ??????

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
......makePodSourceConfig(...)      // pod???manifest, url, apiserver
......NewMainKubelet(...)
......StartGarbageCollection()
....startKubelet(k KubeletBootstrap, podCfg *config.PodConfig, kc *KubeletConfig)
......(kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate)
........(kl *Kubelet) initializeModules() error  // 
......k.ListenAndServe                 // kubelet server
......k.ListenAndServeReadOnly         // kubelet server(ro)
..http.ListenAndServe()  // Healthz Server
```

## ????

| ??                | ??                                   | ??                                       |
| ----------------- | ------------------------------------ | ---------------------------------------- |
| eventBroadcaster  | <- eventBroadcaster.watch.result     | ??Event??EventClient????sink??           |
| containerGC       | 1min                                 |                                          |
| imageGC           | 5min                                 |                                          |
| logServer         |                                      | ?????/logs/                              |
| imageManager      | 5min                                 | ?????????(??????????????)? ???imageManager.imageRecords? |
| containerManager  |                                      | 1. ensureState, 1min 2. ???????RuntimeCgroupsName?KubeletCgroupsName, 5min |
| oomWatcher        | <- ow.cadvisor.WatchEvents().channel | ??OOM Event                              |
| resourceAnalyzer  | 1min                                 | ????pod?????????????                     |
| syncNodeStatus    |                                      |                                          |
| syncNetworkStatus |                                      |                                          |
| updateRuntimeUp   |                                      |                                          |
| podKiller         |                                      |                                          |
| statusManager     |                                      |                                          |
| probeManager      |                                      |                                          |
| pleg              |                                      |                                          |
| syncLoop          |                                      |                                          |
| Healthz           | ???5s??                              | Health server: /healthz                  |



## ????


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