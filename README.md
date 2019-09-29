# kubernetes_hdfs
使用kubernetes部署HDFS

#### 在Kubernetes部署HDFS验证

如果你是Kubernetes专家，那么你可以直接去查看源码[点击这里](https://github.com/hasura/hub-hdfs)

### HDFS的基本架构

让我们首先回顾一下完全分布式HDFS的传统体系结构。下图说明了这种体系结构:

- ![avatar](https://blog.hasura.io/content/images/downloaded_images/getting-started-with-hdfs-on-kubernetes-a75325d4178c/1-5vkJ98W6v2Hg9IMPjZvA2A.png)

  ```
  <center>HDFS架构</center>
  ```

通常，有专用的虚拟机用于运行相应的HDFS守护程序即namenodes和datanodes(我们将使用守护程序和可交替运行守护程序的节点)。Namenodes存储每个文件的元数据，datanodes存储实际的数据。每个HDFS有一个namenode和多个datanodes(大多数情况下考虑高可用，会有两个namenode,但是在演示安装过程中，只考虑一个namenode)

每个守护程序都是用虚拟机的网络相互通信，这点很重要，这将影响我们在Kubernetes上运行它的方式



### HDFS在Kubernetes上的架构

现在，让我们先了解下一个典型的程序在Kubernetes(也就是我们熟知k8s)是怎样的吧。

Kubernetes是节点集群的容器管理器，这意味着它是在集群上部署容器并管理容器的生命周期。Kubernetes中的一种特定资源类型指定了容器的运行方式：是长时间运行还是批处理；是单个实例还是多个副本等等，“pod”是最简单的资源类型，基本上是几个紧密耦合且长期运行的容器的单个实例。

那么，我们的应用程序，一个有不同进程相互交互组成的复杂联合体，在Kubernetes上是什么样子呢？

咋一看，似乎很简单，不同的守护进程将在不同的pod中进行，并使用它们的pod ip相互交互。如下图所示：

![avatar](https://blog.hasura.io/content/images/downloaded_images/getting-started-with-hdfs-on-kubernetes-a75325d4178c/1-zo8kdNokpmDtBh6FFEYNcA.png)

```
       <center>HDFS在kubernetes上的架构</center>
```



不要这么快，你是想将namenode的pod ip直接写在你的datanodes的配置文件里面吗？如果namenode的pod不幸宕机并且在另外的一个服务器中重启（此时pod ip已经发生改变）这该怎么办呢？如果同样的事情发生在datanode上，又该怎么办呢？它们怎么相互联系呢如果他们一直在改变ip，服务器或者是其它在Kubernetes 管理下发生的漂移事件？

#### 所以我们面临如下挑战：

1. Namenodes和datanodes可能宕机并且在另外的pod中重启导致了IP的变化，那它们该如何保持互动的呢？
2. Namenodes和datanodes可能宕机并且在另外的服务器中重启，那么它们正在保存的数据又会发生什么呢？



### 将namenode包装在服务中

k8s解决pod短暂性问题的方法是使用服务资源，Kubernetes服务基本上在集群中为您提供一个静态IP/hostname，来保证对跨选定的pods之间的请求进行负载均衡。pods的选择是基于在pod定义中注入的标签来选择的。所以我们可以给我们的namenode pod一个名为"app:namenode"的标签，然后创建一个带有该标签的pod服务。

这给了我们一个稳定的方式来访问我们的namenode,对于datanodes来说是个好消息。

例如datanode可以在其规范中引用带有服务主机名的namenode

![avatar](https://blog.hasura.io/content/images/downloaded_images/getting-started-with-hdfs-on-kubernetes-a75325d4178c/1-IqsUBQ_ZzAwT6b7sJIRm1g.png)

<center>datanode可以使用k8s服务的hostname</center>

集群中的HDFS客户端可以使用这个hostname

![avatar](https://blog.hasura.io/content/images/downloaded_images/getting-started-with-hdfs-on-kubernetes-a75325d4178c/1-mk6OQZQAiGeAacc0vWqzjg.png)

<center>在集群中使用HDFS客户端</center>

### 通过状态集来标识datanodes

既然我们已经看到了一种与namenode通信的方法，而不用考虑pod行为，那么我们是否可以对datanode做同样的事情呢？

不幸的是，不能。Kubernetes服务的一个基本方面是在它所选择的pods之间随机地进行负载平衡。这就意味着服务选择的所有pods都应该是不可区分的，以便应用程序能够正常运行。我们的datanode并非如此，其中每个datanode都有不同的状态（不同的数据）。datanode看起来更像是“宠物”而不是"牛"

这样有状态的应用非常的常见，因此Kubernetes提供了另一种称为有状态集的资源来帮助此类应用程序。在一个状态集中，每个pod的名称、存储和hostname都有一个粘性标识.瞧！这正是我们想要的。如果我们的每个datanode都始终有它的粘性标识（甚至跨pod重新调度）,那么我们仍然可以确保在应用程序中有稳定的通信和一致的数据。

实际上，如果愿意，我们还可以让我们namenode具有状态。注意，要使所有这些工作正常，我们需要将HDFS配置为使用主机名而不是ip进行通信。这是我们正在使用的docker镜像的默认设置。



### 在单节点上运行完全分布式的HDFS

我们完成了吗?差不多吧，为什么不把所有的工作放在单节点上呢？但是这不是违背了“分布式文件”系统概念吗？不要紧张，在Kubernetes的世界中，分布式的概念是在容器级别。所以，如果我们有多个pods,管理他们的专用磁盘，在一个节点上运行，这就是分布式！

但是对于POC,我们将使用另外的小技巧。我们将模拟单个磁盘上的多个存储卷，并将它们附加到不同的pod上（在更严格的设置中，您将有单独的磁盘支持每个卷，但这只是一个简单的配置问题）。Kubernetes Persistent Volume（PV）资源类型非常适合于此。Kubernetes pv是集群中可用的一组存储卷。一个pod可以请求一个PV，请求成功后将会挂载在pod中。

我们使用类型为"hostPath"的PV来将虚拟服务器的本地目录作为存储块。而且，我们可以用一个具有不同"hostPath"值的磁盘创建任意数量的磁盘。

![avatar](https://blog.hasura.io/content/images/downloaded_images/getting-started-with-hdfs-on-kubernetes-a75325d4178c/1-8OFfzZ8pKDg2u4JOueXtjQ.png)
<center>位于`hostPath`上的PV</center>



