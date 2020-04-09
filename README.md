# Kubernetes-Multus-CNI

## 1. 在 Ubuntu上安装 Kubernetes 集群
具体方式可以见：[ubuntu-Kubernetes](https://github.com/yu3peng/ubuntu-Kubernetes)

## 2. 部署Multus CNI到K8S
下载multus代码
```
git clone https://github.com/intel/multus-cni.git
```

进入image目录，部署Multus环境主要由“flannel-daemonset.yml multus-daemonset.yml ”这两个编排文件完成。flannel-daemonset.yaml部署flannel网络需要的基础组件。Multus在K8S环境中是作为CNI插件使用的，所以和部署一般的网络插件方式类似，由multus-daemonset.yml文件编排完成，主要动作包括权限配置，写入配置到K8S，运行提供CNI功能的容器
```
cd multus-cni/images
cat {flannel-daemonset.yml,multus-daemonset.yml} | kubectl apply -f -
```

在K8S环境中配置额外的CNI插件
```
$ ifconfig | grep ens
ensXXX      Link encap:Ethernet  HWaddr 02:42:ac:11:00:07

vi macvlan-conf-1.yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf-1
spec:
  config: '{
            "cniVersion": "0.3.0",
            "type": "macvlan",
            "master": "ensXXX",
            "mode": "bridge",
            "ipam": {
                "type": "host-local",
                "ranges": [
                    [ {
                         "subnet": "10.10.0.0/16",
                         "rangeStart": "10.10.1.20",
                         "rangeEnd": "10.10.3.50",
                         "gateway": "10.10.0.254"
                    } ]
                ]
            }
        }'

kubectl apply -f macvlan-conf-1.yaml
```

## 3. 使用Multus CNI部署多接口POD
```
vi pod-case-01.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-case-01
  annotations:
    k8s.v1.cni.cncf.io/networks: macvlan-conf-1
spec:
  containers:
  - name: pod-case-01
    image: docker.io/centos/tools:latest
    command:
    - /sbin/init

kubectl apply -f pod-case-01.yaml 
```

部署完成后查看POD接口，eth0为flannel插件提供，net1由macvlan提供
```
$ kubectl get pod
NAME          READY   STATUS    RESTARTS   AGE
pod-case-01   1/1     Running   0          20m
$ kubectl exec pod-case-01 ip a
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if69: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1440 qdisc noqueue state UP group default
    link/ether 56:9b:8e:77:e0:52 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.239.197/32 scope global eth0
       valid_lft forever preferred_lft forever
5: net1@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether 6e:5a:08:a0:da:e0 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.10.1.21/16 scope global net1
       valid_lft forever preferred_lft forever
```
