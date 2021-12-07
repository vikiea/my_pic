# Kubernetes 的架构解析

 **首先，Kubernetes 的官方架构图是这样的。**



![图片](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/11/640-20210928190025952.jpg)



这个架构图看起来会比较复杂，很难看懂，我把这个官方的架构图重新简化了一下，就会非常容易理解了：



![图片](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/11/640.jpg)



- ETCD ：是用来存储所有 Kubernetes 的集群状态的，它除了具备状态存储的功能，还有事件监听和订阅、Leader选举的功能，所谓事件监听和订阅，各个其他组件通信，都并不是互相调用 API 来完成的，而是把状态写入 ETCD（相当于写入一个消息），其他组件通过监听 ETCD 的状态的的变化（相当于订阅消息），然后做后续的处理，然后再一次把更新的数据写入 ETCD。所谓 Leader 选举，其它一些组件比如 Scheduler，为了做实现高可用，通过 ETCD 从多个（通常是3个）实例里面选举出来一个做Master，其他都是Standby。

  

- API Server：刚才说了 ETCD 是整个系统的最核心，所有组件之间通信都需要通过 ETCD，实际上，他们并不是直接访问 ETCD，而是访问一个代理，这个代理是通过标准的RESTFul API，重新封装了对 ETCD 接口调用，除此之外，这个代理还实现了一些附加功能，比如身份的认证、缓存等。这个代理就是 API Server。

  

- Controller Manager：是实现任务的调度的，关于任务调度你可以参考**之前的文章**，简单说，直接请求 Kubernetes 做调度的都是任务，比如比如 Deployment 、Deamon Set 或者 Job，每一个任务请求发送给Kubernetes之后，都是由Controller Manager来处理的，每一个任务类型对应一个Controller Manager，比如 Deployment对应一个叫做 Deployment Controller，DaemonSet 对应一个 DaemonSet Controller。

  

- Scheduler：是用来做资源调度的（具体资源调度的含义请参考**之前的文章**），Controller Manager会把任务对资源要求，其实就是Pod，写入到ETCD里面，Scheduler监听到有新的资源需要调度（新的Pod），就会根据整个集群的状态，给Pod分配到具体的节点上。

  

- Kubelet：是一个Agent，运行在每一个节点上，它会监听ETCD中的Pod信息，发现有分配给它所在节点的Pod需要运行，就在节点上运行相应的Pod，并且把状态更新回到ETCD。

  

- Kubectl: 是一个命令行工具，它会调用 API Server发送请求写入状态到ETCD，或者查询ETCD的状态。 

这样是不是简单了很多。如果还是觉得不清楚，我们就用部署一个服务的例子来解释一个整个过程，假设你要运行一个多个实例的Nginx，那么在Kubernetes内部，整个流程是这样的：

1. 通过kubectl命令行，创建一个包含Nginx的Deployment对象，kubectl会调用 API Server 往ETCD里面写入一个 Deployment对象。
2. Deployment Controller 监听到有新的 Deployment对象被写入，就获取到对象信息，根据对象信息来做任务调度，创建对应的 Replica Set 对象。
3. Replica Set Controller监听到有新的对象被创建，也读取到对象信息来做任务调度，创建对应的Pod来。
4. Scheduler 监听到有新的 Pod 被创建，读取到Pod对象信息，根据集群状态将Pod调度到某一个节点上，然后更新Pod（内部操作是将Pod和节点绑定）。
5. Kubelet 监听到当前的节点被指定了新的Pod，就根据对象信息运行Pod。 



上面就是Kubernetes内部的是如何实现的整个 Deployment 被创建的过程。这个过程只是为了向大家解释每一个组件的职责，以及他们之间是如何相互协作的，忽略掉了很多繁琐的细节。



**Kubernetes 是否是二层调度？**

在 Google 的一篇关于他们内部的 Omega 的调度系统的论文，把调度系统分成三类：单体、二层调度和共享状态三种，按照它的分类方法，通常Google的 Borg被分到单体这一类，Mesos被当做二层调度，而Google自己的Omega被当做第三类“共享状态”。论文的作者实际上之前也是Mesos的设计者之一，后来去了Google设计新的 Omega 系统，并发表了论文，论文的主要目的是提出一种全新的“Shard State”的模式来同时解决调度系统的性能和扩展性的问题，但是实际我觉得 Shared State 模型太过理想化，根据这个模型开发的Omega系统，似乎在Google内部并没有被大规模使用，也没有任何一个大规模使用的调度系统采是采用 Shared State 模型。



![图片](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/11/640-20210928192439778.jpg)



因为Kubernetes的大部分设计是延续 Borg的，而且Kubernetes的核心组件（Controller Manager和Scheduler）缺省也都是绑定部署在一起，状态也都是存储在ETCD里面的的，所以通常大家会把Kubernetes也当做“单体”调度系统，实际上我并不赞同。

我认为 Kubernetes 的调度模型也**完全是二层调度**的，和 Mesos 一样，任务调度和资源的调度是完全分离的，Controller Manager承担任务调度的职责，而Scheduler则承担资源调度的职责。 

![图片](https://gitee.com/vikieq/my_pic/raw/master/uPic/2021/10/11/640-20210928192439807.jpg)



实际上Kubernetes和Mesos调度的最大区别在于资源调度请求的方式：



- 主动 Push 方式。是 Mesos 采用的方式，就是 Mesos 的资源调度组件（Mesos Master）主动推送资源 Offer 给 Framework，Framework 不能主动请求资源，只能根据 Offer 的信息来决定接受或者拒绝。

  

- 被动 Pull 方式。是 Kubernetes 的方式，资源调度组件 Scheduler 被动的响应 Controller Manager的资源请求。