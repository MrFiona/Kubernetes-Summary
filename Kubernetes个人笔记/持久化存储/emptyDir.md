# emptyDir

emptyDir 是最基础的 Volume 类型。正如其名字所示，一个 emptyDir Volume 是 Host 上的一个空目录。

emptyDir Volume 对于容器来说是持久的，对于 Pod 则不是。当 Pod 从节点删除时，Volume 的内容也会被删除。但如果只是容器被销毁而 Pod 还在，则 Volume 不受影响。

也就是说：emptyDir Volume 的生命周期与 Pod 一致。

Pod 中的所有容器都可以共享 Volume，它们可以指定各自的 mount 路径。下面通过例子来实践 emptyDir，配置文件如下：

![存储-1](/assets/存储-1.PNG)