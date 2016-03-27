
最后调用的是kubelet和kube-proxy

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