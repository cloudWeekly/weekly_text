
## k8s的架构中需要通过SSL认证的地方

![image](https://user-images.githubusercontent.com/12036324/69598284-d031cf80-1043-11ea-8283-ff623ace9829.png)

上图显示了k8s各个组件之间的调用关系，在安全认证的环境下一般都是要通过证书认证的。

证书可以分为两种：客户端（client auth）和服务端（server auth）：
### master节点

- ETCD：即需要客户端证书也要服务端证书
    + 作为api-server的服务端
    + 如果是集群的话，也作为客户端去连集群
- api-server：即需要客户端证书也要服务端证书
    + ETCD的客户端
    + 其他组件的服务端
- controller-manager: 通过非安全的8080和apiserver进行通信
- schedule：通过非安全的8080和apiserver进行通信
- kubectl：只要客户端证书

为什么contoller-manager和sceduler通过非安全模式访问： [schedule 和kube-controller-manager 配置 127.0.0.1:8080问题](https://github.com/easzlab/kubeasz/issues/397)
### node节点

- kubelet: 即需要客户端证书也要服务端证书
    + 二进制部署要表示自己的服务
    + 作为apiserver的客户端
- kube-proxy：只需要客户端证书
- 网络插件（flannel、calico）：只需要客户端证书




解释一下证书可能遇到的几个文件的含义：
https://blog.51cto.com/liuzhengwei521/2120535



