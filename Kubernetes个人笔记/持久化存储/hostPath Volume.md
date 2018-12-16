# hostPath Volume

hostPath Volume 的作用是将 Docker Host 文件系统中已经存在的目录 mount 给 Pod 的容器。大部分应用都不会使用 hostPath Volume，因为这实际上增加了 Pod 与节点的耦合，限制了 Pod 的使用。不过那些需要访问 Kubernetes 或 Docker 内部数据（配置文件和二进制库）的应用则需要使用 hostPath。

比如 kube-apiserver 和 kube-controller-manager 就是这样的应用，通过

`kubectl edit --namespace=kube-system pod kube-apiserver-k8s-master`