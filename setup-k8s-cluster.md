# 如何手动搭建k8s集群

## 0. 介绍



## 1. kubelet
[/etc/default/kubelet](scripts/setup-k8s-cluster/kubelet.default)  
[/etc/init.d/kubelet](scripts/setup-k8s-cluster/kubelet)  
/usr/local/bin/kubelet

## etcd
**/etc/kubernetes/manifests/etcd.manifest**
[/etc/kubernetes/manifests/etcd.manifest](scripts/setup-k8s-cluster/etcd.manifest) 
## apiserver
**/etc/kubernetes/manifests/apiserver.manifest**
[/etc/kubernetes/manifests/apiserver.manifest](scripts/setup-k8s-cluster/apiserver.manifest) 
## controller-manager
**/etc/kubernetes/manifests/controller-manager.manifest**
[/etc/kubernetes/manifests/controller-manager.manifest](scripts/setup-k8s-cluster/controller-manager.manifest) 
## scheduler
**/etc/kubernetes/manifests/scheduler.manifest**
[/etc/kubernetes/manifests/scheduler.manifest](scripts/setup-k8s-cluster/scheduler.manifest) 

## 参数说明

```sh
kubelet --api-servers=http://127.0.0.1:8080 --config=/etc/kubernetes/manifests --v=2 --allow-privileged=true

etcd --listen-peer-urls http://127.0.0.1:2380 --addr 127.0.0.1:4001 --bind-addr 127.0.0.1:4001 --data-dir /var/etcd/data

kube-apiserver --insecure-bind-address=0.0.0.0 --insecure-port=8080 --service-cluster-ip-range=10.254.0.0/16 --log_dir=/var/log/kube  --logtostderr=true --etcd-servers=http://127.0.0.1:4001 --allow_privileged=false

kube-controller-manager --master=127.0.0.1:8080

kube-scheduler --master=127.0.0.1:8080
```

### kubelet

| 参数          | 说明   |
| ----------- | ---- |
| api-servers |      |
| config      |      |
| v           |      |

### etcd

| 参数               | 说明   |
| ---------------- | ---- |
| listen-peer-urls |      |
| addr             |      |
| bind-addr        |      |
| data-dir         |      |

### apiserver

| 参数                       | 说明   |
| ------------------------ | ---- |
| insecure-bind-address    |      |
| insecure-port            |      |
| service-cluster-ip-range |      |
| log_dir                  |      |
| logtostderr              |      |
| etcd-servers             |      |
| allow_privileged         |      |
|                          |      |

### controller-manager

| 参数     | 说明   |
| ------ | ---- |
| master |      |

## scheduler
| 参数     | 说明   |
| ------ | ---- |
| master |      |