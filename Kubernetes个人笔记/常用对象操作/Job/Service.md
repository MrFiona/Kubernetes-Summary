# Service

我们前面的课程中学习了`Pod`的基本用法，我们也了解到`Pod`的生命是有限的，死亡过后不会复活了。我们后面学习到的`RC`和`Deployment`可以用来动态的创建和销毁`Pod`。尽管每个`Pod`都有自己的`IP`地址，但是如果`Pod`重新启动了的话那么他的`IP`很有可能也就变化了。这就会带来一个问题：比如我们有一些后端的`Pod`的集合为集群中的其他前端的`Pod`集合提供`API`服务，如果我们在前端的`Pod`中把所有的这些后端的`Pod`的地址都写死，然后去某种方式去访问其中一个`Pod`的服务，这样看上去是可以工作的，对吧？但是如果这个`Pod`挂掉了，然后重新启动起来了，是不是`IP`地址非常有可能就变了，这个时候前端就极大可能访问不到后端的服务了。

遇到这样的问题该怎么解决呢？在没有使用`Kubernetes`之前，我相信可能很多同学都遇到过这样的问题，不一定是`IP`变化的问题，比如我们在部署一个`WEB`服务的时候，前端一般部署一个`Nginx`作为服务的入口，然后`Nginx`后面肯定就是挂载的这个服务的大量后端，很早以前我们可能是去手动更改`Nginx`配置中的`upstream`选项，来动态改变提供服务的数量，到后面出现了一些`服务发现`的工具，比如`Consul`、`ZooKeeper`还有我们熟悉的`etcd`等工具，有了这些工具过后我们就可以只需要把我们的服务注册到这些服务发现中心去就可以，然后让这些工具动态的去更新`Nginx`的配置就可以了，我们完全不用去手工的操作了，是不是非常方便。
![nginx](/assets/nginx-consul.png)

同样的，要解决我们上面遇到的问题是不是实现一个服务发现的工具也可以解决啊？没错的，当我们`Pod`被销毁或者新建过后，我们可以把这个`Pod`的地址注册到这个服务发现中心去就可以，但是这样的话我们的前端的`Pod`结合就不能直接去连接后台的`Pod`集合了是吧，应该连接到一个能够做服务发现的中间件上面，对吧？

没错，`Kubernetes`集群就为我们提供了这样的一个对象 - `Service`，`Service`是一种抽象的对象，它定义了一组`Pod`的逻辑集合和一个用于访问它们的策略，其实这个概念和微服务非常类似。一个`Serivce`下面包含的`Pod`集合一般是由`Label Selector`来决定的。

比如我们上面的例子，假如我们后端运行了3个副本，这些副本都是可以替代的，因为前端并不关心它们使用的是哪一个后端服务。尽管由于各种原因后端的`Pod`集合会发送变化，但是前端却不需要知道这些变化，也不需要自己用一个列表来记录这些后端的服务，`Service`的这种抽象就可以帮我们达到这种解耦的目的。


## 三种IP
在继续往下学习`Service`之前，我们需要先弄明白`Kubernetes`系统中的三种IP这个问题，因为经常有同学混乱。

* Node IP：`Node`节点的`IP`地址
* Pod IP: `Pod`的IP地址
* Cluster IP: `Service`的`IP`地址

首先，`Node IP`是`Kubernetes`集群中节点的物理网卡`IP`地址(一般为内网)，所有属于这个网络的服务器之间都可以直接通信，所以`Kubernetes`集群外要想访问`Kubernetes`集群内部的某个节点或者服务，肯定得通过`Node IP`进行通信（这个时候一般是通过外网`IP`了）

然后`Pod IP`是每个`Pod`的`IP`地址，它是`Docker Engine`根据`docker0`网桥的`IP`地址段进行分配的（我们这里使用的是`flannel`这种网络插件保证所有节点的`Pod IP`不会冲突）

最后`Cluster IP`是一个虚拟的`IP`，仅仅作用于`Kubernetes Service`这个对象，由`Kubernetes`自己来进行管理和分配地址，当然我们也无法`ping`这个地址，他没有一个真正的实体对象来响应，他只能结合`Service Port`来组成一个可以通信的服务。


## 定义Service

定义`Service`的方式和我们前面定义的各种资源对象的方式类型，例如，假定我们有一组`Pod`服务，它们对外暴露了 8080 端口，同时都被打上了`app=myapp`这样的标签，那么我们就可以像下面这样来定义一个`Service`对象：

```yaml
apiVersion: v1
kind: Service
metadata: 
  name: myservice
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
    name: myapp-http
```
然后通过的使用`kubectl create -f myservice.yaml`就可以创建一个名为`myservice`的`Service`对象，它会将请求代理到使用 TCP 端口为 8080，具有标签`app=myapp`的`Pod`上，这个`Service`会被系统分配一个我们上面说的`Cluster IP`，该`Service`还会持续的监听`selector`下面的`Pod`，会把这些`Pod`信息更新到一个名为`myservice`的`Endpoints`对象上去，这个对象就类似于我们上面说的`Pod`集合了。

需要注意的是，`Service`能够将一个接收端口映射到任意的`targetPort`。 默认情况下，`targetPort`将被设置为与`port`字段相同的值。 可能更有趣的是，targetPort 可以是一个字符串，引用了 backend Pod 的一个端口的名称。 因实际指派给该端口名称的端口号，在每个 backend Pod 中可能并不相同，所以对于部署和设计 Service ，这种方式会提供更大的灵活性。 

另外`Service`能够支持 TCP 和 UDP 协议，默认是 TCP 协议。


## kube-proxy

前面我们讲到过，在`Kubernetes`集群中，每个`Node`会运行一个`kube-proxy`进程, 负责为`Service`实现一种 VIP（虚拟 IP，就是我们上面说的`clusterIP`）的代理形式，现在的`Kubernetes`中默认是使用的`iptables`这种模式来代理。这种模式，`kube-proxy`会监视`Kubernetes master`对 Service 对象和 Endpoints 对象的添加和移除。 对每个 Service，它会添加上 iptables 规则，从而捕获到达该 Service 的 clusterIP（虚拟 IP）和端口的请求，进而将请求重定向到 Service 的一组 backend 中的某一个个上面。 对于每个 Endpoints 对象，它也会安装 iptables 规则，这个规则会选择一个 backend Pod。

默认的策略是，随机选择一个 backend。 我们也可以实现基于客户端 IP 的会话亲和性，可以将 `service.spec.sessionAffinity` 的值设置为 "ClientIP" （默认值为 "None"）。

另外需要了解的是如果最开始选择的 Pod 没有响应，iptables 代理能够自动地重试另一个 Pod，所以它需要依赖 readiness probes。

![service iptables overview](/assets/services-iptables-overview.PNG)


## Service 类型

我们在定义`Service`的时候可以指定一个自己需要的类型的`Service`，如果不指定的话默认是`ClusterIP`类型。

我们可以使用的服务类型如下：

* ClusterIP：通过集群的内部 IP 暴露服务，选择该值，服务只能够在集群内部可以访问，这也是默认的ServiceType。

* NodePort：通过每个 Node 节点上的 IP 和静态端口（NodePort）暴露服务。NodePort 服务会路由到 ClusterIP 服务，这个 ClusterIP 服务会自动创建。通过请求 <NodeIP>:<NodePort>，可以从集群的外部访问一个 NodePort 服务。
* LoadBalancer：使用云提供商的负载局衡器，可以向外部暴露服务。外部的负载均衡器可以路由到 NodePort 服务和 ClusterIP 服务，这个需要结合具体的云厂商进行操作。
* ExternalName：通过返回 CNAME 和它的值，可以将服务映射到 externalName 字段的内容（例如， foo.bar.example.com）。没有任何类型代理被创建，这只有 Kubernetes 1.7 或更高版本的 kube-dns 才支持。

### NodePort 类型

如果设置 type 的值为 "NodePort"，Kubernetes master 将从给定的配置范围内（默认：30000-32767）分配端口，每个 Node 将从该端口（每个 Node 上的同一端口）代理到 Service。该端口将通过 Service 的 spec.ports[*].nodePort 字段被指定，如果不指定的话会自动生成一个端口。

需要注意的是，Service 将能够通过 <NodeIP>:spec.ports[*].nodePort 和 spec.clusterIp:spec.ports[*].port 而对外可见。

接下来我们来给大家创建一个`NodePort`的服务来访问我们前面的`Nginx`服务：(保存为service-demo.yaml)
```yaml
apiVersion: v1
kind: Service
metadata: 
  name: myservice
spec:
  selector:
    app: myapp
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    name: myapp-http
```

创建该`Service`:
```
$ kubectl create -f service-demo.yaml
```

然后我们可以查看`Service`对象信息：
```
$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        27d
myservice    NodePort    10.104.57.198   <none>        80:32560/TCP   14h
```
我们可以看到`myservice`的 TYPE 类型已经变成了`NodePort`，后面的`PORT(S)`部分也多了一个 32560 的映射端口。

### ExternalName

`ExternalName` 是 Service 的特例，它没有 selector，也没有定义任何的端口和 Endpoint。 对于运行在集群外部的服务，它通过返回该外部服务的别名这种方式来提供服务。
```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

当查询主机 my-service.prod.svc.cluster.local （后面服务发现的时候我们会再深入讲解）时，集群的 DNS 服务将返回一个值为 my.database.example.com 的 CNAME 记录。 访问这个服务的工作方式与其它的相同，唯一不同的是重定向发生在 DNS 层，而且不会进行代理或转发。 如果后续决定要将数据库迁移到 Kubernetes 集群中，可以启动对应的 Pod，增加合适的 Selector 或 Endpoint，修改 Service 的 type，完全不需要修改调用的代码，这样就完全解耦了。


### 通过 Service 访问 Pod

Kubernetes Service 从逻辑上代表了一组 Pod，具体是哪些 Pod 则是由 label 来挑选。Service 有自己 IP，而且这个 IP 是不变的。客户端只需要访问 Service 的 IP，Kubernetes 则负责建立和维护 Service 与 Pod 的映射关系。无论后端 Pod 如何变化，对客户端不会有任何影响，因为 Service 没有变。

来看个例子，创建下面的这个 Deployment：

![创建Deployment](/assets/640.webp)

我们启动了三个 Pod，运行 httpd 镜像，label 是 run: httpd，Service 将会用这个 label 来挑选 Pod。

![挑选pod](/assets/640-1.webp)

Pod 分配了各自的 IP，这些 IP 只能被 Kubernetes Cluster 中的容器和节点访问。

![挑选pod](/assets/640-2.webp)

接下来创建 Service，其配置文件如下：

![挑选pod](/assets/640-3.webp)

① v1 是 Service 的 apiVersion。

② 指明当前资源的类型为 Service。

③ Service 的名字为 httpd-svc。

④ selector 指明挑选那些 label 为 run: httpd 的 Pod 作为 Service 的后端。

⑤ 将 Service 的 8080 端口映射到 Pod 的 80 端口，使用 TCP 协议。

执行 kubectl apply 创建 Service httpd-svc。

![挑选pod](/assets/640-4.webp)

httpd-svc 分配到一个 CLUSTER-IP 10.99.229.179。可以通过该 IP 访问后端的 httpd Pod。

![挑选pod](/assets/640-5.webp)

根据前面的端口映射，这里要使用 8080 端口。另外，除了我们创建的 httpd-svc，还有一个 Service kubernetes，Cluster 内部通过这个 Service 访问 kubernetes API Server。

通过 kubectl describe 可以查看 httpd-svc 与 Pod 的对应关系。

![挑选pod](/assets/640-6.webp)

Endpoints 罗列了三个 Pod 的 IP 和端口。我们知道 Pod 的 IP 是在容器中配置的，那么 Service 的 Cluster IP 又是配置在哪里的呢？CLUSTER-IP 又是如何映射到 Pod IP 的呢？

答案是 iptables，我们下节讨论。

### Service IP 原理

Service Cluster IP 是一个虚拟 IP，是由 Kubernetes 节点上的 iptables 规则管理的。

可以通过 iptables-save 命令打印出当前节点的 iptables 规则，因为输出较多，这里只截取与 httpd-svc Cluster IP 10.99.229.179 相关的信息：

![挑选pod](/assets/service1.PNG)

这两条规则的含义是：

  - 如果 Cluster 内的 Pod（源地址来自 172.254.0.0/16）要访问 shb-sf-stg-4c7c8873/anbot-chat，则允许。

  - 其他源地址访问 shb-sf-stg-4c7c8873/anbot-chat，跳转到规则 KUBE-SVC-PG7U6NWYNWEVV50B。

KUBE-SVC-PG7U6NWYNWEVV50B 规则如下：

![挑选pod](/assets/service2.PNG)

- 1/2 的概率跳转到规则 KUBE-SEP-NXUPZSXECK45VXWX。
- 1/2 的概率跳转到规则 KUBE-SEP-NFFTFQ5CNTTL2FEL。

上面两个个跳转的规则如下：

![挑选pod](/assets/service3.PNG)

即将请求分别转发到后端的两个 Pod。通过上面的分析，我们得到如下结论：

iptables 将访问 Service 的流量转发到后端 Pod，而且使用类似轮询的负载均衡策略。另外需要补充一点：Cluster 的每一个节点都配置了相同的 iptables 规则，这样就确保了整个 Cluster 都能够通过 Service 的 Cluster IP 访问 Service。

![挑选pod](/assets/service-4.PNG)

![挑选pod](/assets/service-5.PNG)

### 外网访问 Service

除了 Cluster 内部可以访问 Service，很多情况我们也希望应用的 Service 能够暴露给 Cluster 外部。Kubernetes 提供了多种类型的 Service，默认是 ClusterIP。

#### ClusterIP 
Service 通过 Cluster 内部的 IP 对外提供服务，只有 Cluster 内的节点和 Pod 可访问，这是默认的 Service 类型，前面实验中的 Service 都是 ClusterIP。

#### NodePort 
Service 通过 Cluster 节点的静态端口对外提供服务。Cluster 外部可以通过 <NodeIP>:<NodePort> 访问 Service。

#### LoadBalancer 
Service 利用 cloud provider 特有的 load balancer 对外提供服务，cloud provider 负责将 load balancer 的流量导向 Service。目前支持的 cloud provider 有 GCP、AWS、Azur 等。

下面我们来实践 NodePort，Service httpd-svc 的配置文件修改如下：

![挑选pod](/assets/service-6.PNG)

添加 type: NodePort，重新创建 httpd-svc。

![挑选pod](/assets/service-7.PNG)

Kubernetes 依然会为 httpd-svc 分配一个 ClusterIP，不同的是：

- EXTERNAL-IP 为 nodes，表示可通过 Cluster 每个节点自身的 IP 访问 Service。

- PORT(S) 为 8080:32312。8080 是 ClusterIP 监听的端口，32312 则是节点上监听的端口。Kubernetes 会从 30000-32767 中分配一个可用的端口，每个节点都会监听此端口并将请求转发给 Service。

![挑选pod](/assets/service-8.PNG)

下面测试 NodePort 是否正常工作。

![挑选pod](/assets/service-9.PNG)

通过三个节点 IP + 32312 端口都能够访问 httpd-svc。

Kubernetes 是如何将 `<NodeIP>:<NodePort>` 映射到 Pod 的呢？

与 ClusterIP 一样，也是借助了 iptables。与 ClusterIP 相比，每个节点的 iptables 中都增加了下面两条规则：

![挑选pod](/assets/service-10.PNG)

规则的含义是：访问当前节点 32312 端口的请求会应用规则 KUBE-SVC-RL3JAE4GN7VOGDGP，内容为：

![挑选pod](/assets/service-11.PNG)

其作用就是负载均衡到每一个 Pod。

NodePort 默认是的随机选择，不过我们可以用 nodePort 指定某个特定端口。

![挑选pod](/assets/service-12.PNG)

现在配置文件中就有三个 Port 了：
nodePort 是节点上监听的端口。
port 是 ClusterIP 上监听的端口。
targetPort 是 Pod 监听的端口即容器的服务端口。

最终，Node 和 ClusterIP 在各自端口上接收到的请求都会通过 iptables 转发到 Pod 的 targetPort。

应用新的 nodePort 并验证：

![挑选pod](/assets/service--13.PNG)

nodePort: 30000 已经生效了。


