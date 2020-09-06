
## 一、k8s中DNS的演进

### v1.2 - v1.4之前
有四个容器：
- kube2sky：监控k8s中service的变化，根据service名称和IP生成DNS记录保存在ETCD里面
- ETCD：pod内的存储
- skydns：为POD提供查询服务
- healthz：提供对skydns服务的健康检查

![image](https://user-images.githubusercontent.com/12036324/69865384-00939b00-12dc-11ea-83df-e481588925f2.png)


### v1.4 - v1.11之前

用kubeDNS代替skyDNS组件，考虑到的是SkyDNS组件之间的通信比较多，整体性能不高。所以kubeDNS由下面三个容器组成：
- kubedns： 监控k8s中service的变化，根据service名称和IP地址生成DNS记录，**并且将DNS记录保存在内存中**
- dnsmasq：从kubedns中获取DNS记录，提供DNS缓存，为客户端容器提供DNS的查询服务(kubeDNS 模式下，dnsmasq 在内存中预留一块大小（默认是 1G）的地方，保存当前最常用的 DNS 查询记录，如果缓存中没有要查找的记录，它会到 kubeDNS 中查询，并把结果缓存起来)
- sidecar：提供对kube-dns和dnsmasq服务的健康检查功能
![image](https://user-images.githubusercontent.com/12036324/69865759-18b7ea00-12dd-11ea-9ea4-4359b437335b.png)

### 1.11 - now


## k8s的服务发现

### 0. 如何生效
在启动kubelet的时候指定下面两个参数：
- --cluster-dns： dns在集群中的vip
- --cluster_domain： 域名的后缀

这样每次kubelet启动pod的时候都会注入进去了

### 1. k8s DNS策略

k8s中POD的DNS策略有四种类型：
- Default： pod继承所在主机上的DNS配置
- ClusterFirst（默认）：先在k8s集群配置的coreDNS中查询，查不到再去继承主机的上游nameserver中查询
- ClusterFirstWithHostNet：对于网络配置为hostNetwork的pod而言，其DNS配置规则与ClusterFirst一致
- None：忽略k8s环境的DNS配置，只认pod的dnsConfig设置

### 2. resolv.conf文件解析
*resolv.conf*文件一般有三行，
```shell
# k exec -ti pod-name  -n develop cat /etc/resolv.conf
nameserver 10.42.0.10
search develop.svc.cluster.local. svc.cluster.local. cluster.local. helios.io
options ndots:5
```
#### nameserver
指定的就是DNS服务的IP，在集群中就是就是DNS的svc的ip：
```shell
# k get svc -n kube-system | grep dns
kube-dns               ClusterIP   10.42.0.10     <none>        53/UDP,53/TCP   1d
```

所有的域名解析，都要经过dns的虚ip解析。

#### search

在解析域名的时候，将要访问的域名依次带入search域，进行DNS查询。


比如在pod中访问一个域名为world的服务，其DNS域名查询的顺序为：
```shell
world.develop.svc.cluster.local. world.svc.cluster.local. world.cluster.local. world.helios.io
```

#### options

第三行是可选项，最常用的就是dnots。比如： *options ndots:5*。
- 如果dnots包含的“.”小于5，则先走search，在用绝对域名
- 如果查询的域名“.”大于等于5，则先用绝对域名再走search域


如果我访问的是 a.b.c.e.f.g ，那么域名查找的顺序如下：
```shell
a.b.c.e.f.g. -> a.b.c.e.f.g.default.svc.cluster.local. -> a.b.c.e.f.g.svc.cluster.local. -> a.b.c.e.f.g.cluster.local.
```
如果我访问的是 a.b.c.e，那么域名查找的顺序如下：
```shell
a.b.c.e.default.svc.cluster.local. -> a.b.c.e.svc.cluster.local. -> a.b.c.e.cluster.local. -> a.b.c.e.
```

### 3. 访问外部地址

看完上面的，我们会发现，如果那如果在上面的配置中，我访问baidu.com那不是先给走到了search域了。

答案是肯定的，根据上面的配置，访问baidu.com的走的路径也就是：
```shell
baidu.com.develop.svc.cluster.local. baidu.com.svc.cluster.local. baidu.com.cluster.local. baidu.com.helios.io baidu.com
```
看吧，白白浪费了四次的DNS查询。







## 介绍


## 创建

## 如何实现服务发现·


## 参考：

### 文档类：
- [CoreDNS GA for Kubernetes Cluster DNS](https://kubernetes.io/blog/2018/07/10/coredns-ga-for-kubernetes-cluster-dns/)
- [dns#how-do-i-test-if-it-is-working](https://github.com/kubernetes/kubernetes/tree/release-1.2/cluster/addons/dns#how-do-i-test-if-it-is-working)
- [DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pods-dns-policy)
- [Customizing DNS Service](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)
- [dnsmasq (简体中文)](https://wiki.archlinux.org/index.php/Dnsmasq_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

### issue:

- [KubeDNS endpoint shows status failure ](https://github.com/kubernetes/kubernetes/issues/38896)
- [kube-dns copies 127.0.0.35 from host's /etc/resolv.conf, doesn't work](https://github.com/kubernetes/kubernetes/issues/45828)
- [Latest Build - DNS Does not Resolve ](https://github.com/kelseyhightower/kubernetes-the-hard-way/issues/356)
- [kube-dns 容器不断重启](https://github.com/easzlab/kubeasz/issues/159)
- [Install on a system using `systemd-resolved` leads to broken DNS](https://github.com/kubernetes/kubeadm/issues/273)
- [skydns stops working randomly ](https://github.com/kubernetes/kubernetes/issues/22161)


### blog:
- [CoreDNS 与 127.0.0.53 的仇恨](https://www.twblogs.net/a/5c15e9e3bd9eee5e418424b6/zh-cn)
- [systemd-resolved.service 中文手册](http://www.jinbuguo.com/systemd/systemd-resolved.service.html)

