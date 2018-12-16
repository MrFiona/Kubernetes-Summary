# emptyDir

首先我们会学习 Volume，以及 Kubernetes 如何通过 Volume 为集群中的容器提供存储；然后我们会实践几种常用的 Volume 类型并理解它们各自的应用场景；最后，我们会讨论 Kubernetes 如何通过 Persistent Volume 和 Persistent Volume Claim 分离集群管理员与集群用户的职责，并实践 Volume 的静态供给和动态供给。

Volume
本节我们讨论 Kubernetes 的存储模型 Volume，学习如何将各种持久化存储映射到容器。

我们经常会说：容器和 Pod 是短暂的。
其含义是它们的生命周期可能很短，会被频繁地销毁和创建。容器销毁时，保存在容器内部文件系统中的数据都会被清除。

为了持久化保存容器的数据，可以使用 Kubernetes Volume。

Volume 的生命周期独立于容器，Pod 中的容器可能被销毁和重建，但 Volume 会被保留。

本质上，Kubernetes Volume 是一个目录，这一点与 Docker Volume 类似。当 Volume 被 mount 到 Pod，Pod 中的所有容器都可以访问这个 Volume。Kubernetes Volume 也支持多种 backend 类型，包括 emptyDir、hostPath、GCE Persistent Disk、AWS Elastic Block Store、NFS、Ceph 等，完整列表可参考 https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes

Volume 提供了对各种 backend 的抽象，容器在使用 Volume 读写数据的时候不需要关心数据到底是存放在本地节点的文件系统中呢还是云硬盘上。对它来说，所有类型的 Volume 都只是一个目录。

我们将从最简单的 emptyDir 开始学习 Kubernetes Volume。