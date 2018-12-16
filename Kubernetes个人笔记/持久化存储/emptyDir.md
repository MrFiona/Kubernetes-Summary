# emptyDir

emptyDir 是最基础的 Volume 类型。正如其名字所示，一个 emptyDir Volume 是 Host 上的一个空目录。

emptyDir Volume 对于容器来说是持久的，对于 Pod 则不是。当 Pod 从节点删除时，Volume 的内容也会被删除。但如果只是容器被销毁而 Pod 还在，则 Volume 不受影响。

也就是说：emptyDir Volume 的生命周期与 Pod 一致。

Pod 中的所有容器都可以共享 Volume，它们可以指定各自的 mount 路径。下面通过例子来实践 emptyDir，配置文件如下：

![存储-1](/assets/存储1.PNG)

这里我们模拟了一个 producer-consumer 场景。Pod 有两个容器 producer和 consumer，它们共享一个 Volume。producer 负责往 Volume 中写数据，consumer 则是从 Volume 读取数据。

① 文件最底部 volumes 定义了一个 emptyDir 类型的 Volume shared-volume。

② producer 容器将 shared-volume mount 到 /producer_dir 目录。

③ producer 通过 echo 将数据写到文件 hello 里。

④ consumer 容器将 shared-volume mount 到 /consumer_dir 目录。

⑤ consumer 通过 cat 从文件 hello 读数据。

执行如下命令创建 Pod：

![存储-1](/assets/存储2.PNG)

kubectl logs 显示容器 consumer 成功读到了 producer 写入的数据，验证了两个容器共享 emptyDir Volume。

因为 emptyDir 是 Docker Host 文件系统里的目录，其效果相当于执行了 docker run -v /producer_dir 和 docker run -v /consumer_dir。通过 docker inspect 查看容器的详细配置信息，我们发现两个容器都 mount 了同一个目录：

![存储-1](/assets/存储3.PNG)

![存储-1](/assets/存储4.PNG)

这里 /var/lib/kubelet/pods/3e6100eb-a97a-11e7-8f72-0800274451ad/volumes/kubernetes.io~empty-dir/shared-volume 就是 emptyDir 在 Host 上的真正路径。

emptyDir 是 Host 上创建的临时目录，其优点是能够方便地为 Pod 中的容器提供共享存储，不需要额外的配置。但它不具备持久性，如果 Pod 不存在了，emptyDir 也就没有了。根据这个特性，emptyDir 特别适合 Pod 中的容器需要临时共享存储空间的场景，比如前面的生产者消费者用例。
