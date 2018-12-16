# hostPath Volume

hostPath Volume 的作用是将 Docker Host 文件系统中已经存在的目录 mount 给 Pod 的容器。大部分应用都不会使用 hostPath Volume，因为这实际上增加了 Pod 与节点的耦合，限制了 Pod 的使用。不过那些需要访问 Kubernetes 或 Docker 内部数据（配置文件和二进制库）的应用则需要使用 hostPath。

比如 kube-apiserver 和 kube-controller-manager 就是这样的应用，通过

`kubectl edit --namespace=kube-system pod kube-apiserver-k8s-master`

![存储-1](/assets/存储5.PNG)

这里定义了三个 hostPath volume k8s、certs 和 pki，分别对应 Host 目录 /etc/kubernetes、/etc/ssl/certs 和 /etc/pki。

如果 Pod 被销毁了，hostPath 对应的目录也还会被保留，从这点看，hostPath 的持久性比 emptyDir 强。不过一旦 Host 崩溃，hostPath 也就没法访问了。