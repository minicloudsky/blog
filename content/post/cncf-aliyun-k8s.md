---
categories:
- k8a
date: 2023-05-14 21:35:12
tags:
- 思考
title: cncf & aliyun k8s笔记
---

## K8s核心概念
* 调度
*  自动修复
*  水平伸缩

### k8s架构
典型的两层client-server架构
###  master 核心组件
* API Server
* Controller
* Scheduler
*  etcd

<!--more-->

### Pod
容器的本质实际上是一个进程，是一个视图被隔离，资源受限的进程

* k8s -> 云操作系统
* 容器镜像 -> 云操作系统安装包

### 进程组
容器是单进程模型，管理容器就是管理进程，一个程序有多个进程

pod = "进程组"

一个应用的多个进程定义为多个容器启动起来，定义在一个pod里面

Pod 在 Kubernetes 里面只有一个逻辑单位，没有一个真实的东西对应说这个就是 Pod，真正起来在物理上存在的东西，就是四个容器。

### Pod -> K8s原子调度单位 Why?

* 容器之间紧密协作，有依赖Task co-scheduling关系

### 超亲密关系
* 亲密关系：调度解决
  * 两个应用需要运行在同一个宿主机上
* 超亲密关系: pod解决
	* 会发生直接的文件交换
	* localhost或者socket本地通信
	* 频繁的RPC调用
	* 共享linux namespace(etc,一个容器要加入另一个容器的network namespace)
	* ...

### Pod 实现
* 共享网络
	* infra container，整个pod内的container共享同一个网络，其他所有容器加入infra container的network namespace，启动时候infra container 先启动，整个 Pod 的生命周期是等同于 Infra container 的生命周期
* 共享存储
	* pod内多个容器挂载同一个volume

### 容器设计模式
Init Container 按照定义顺序做初始化工作

### Sidecar
在 Pod 里面，可以定义一些专门的容器，来执行主业务容器所需要的一些辅助工作
* ssh
* 日志收集
* debug
辅助功能解耦业务，可以独立发布sidecar容器

### Sidecar 代理容器
* 代理容器对业务容器屏蔽被代理的服务集群，简化业务代码的实现逻辑
	* 容器之间通过localhost直接访问
	* 代理容器的代码可以对全公司复用

### Sidecar适配器容器

### 设计模式本质	: 解耦和复用

## 应用编排管理  
1. k8s资源元信息
	* Spec: 期望状态
	* Status: 观测到的状态
	* Labels: 资源注解
	* Annotations: 多个资源之间相互关系的 OwnerReference
2. Labels
	* key: value元数据
	* 筛选资源，唯一的组合资源方法
	* 集合型selector
3.  Annotations
	* 系统或者工具用来存储资源的非标示性信息
	* 扩展资源的 spec/status 的描述
	* 特点
		* 一般比label更大
		* 可以包含特殊字符
		* 可以结构化也可以非结构化
4. Ownerreference
	* 所有者，一般就是指集合类的资源
	* 集合类资源的控制器会创建对应的归属资源。比如：replicaset 控制器在操作中会创建 Pod，被创建 Pod 的 Ownereference 就指向了创建 Pod 的 replicaset，Ownereference 使得用户可以方便地查找一个创建资源的对象，另外，还可以用来实现级联删除的效果。

### 控制器模式
1. 控制循环
	* 控制器
	* 被控制的系统
	* 能跟观测系统的传感器

	外界通过修改资源 spec 来控制资源，控制器比较资源 spec 和 status，从而计算一个 diff，diff 最后会用来决定执行对系统进行什么样的控制操作，控制操作会使得系统产生新的输出，并被传感器以资源 status 形式上报，控制器的各个组件将都会是独立自主地运行，不断使系统向 spec 表示终态趋近。
	
	各组件独立自主运行，不断使系统向终态趋近 status -> spec
2. Sensor
逻辑传感器
* Reflector
* Informer
* Indexer

Reflector 通过 List 和 Watch K8s server 来获取资源的数据。List 用来在 Controller 重启以及 Watch 中断的情况下，进行系统资源的全量更新；而 Watch 则在多次 List 之间进行增量的资源更新；Reflector 在获取新的资源数据后，会在 Delta 队列中塞入一个包括资源对象信息本身以及资源对象事件类型的 Delta 记录，Delta 队列中可以保证同一个对象在队列中仅有一条记录，从而避免 Reflector 重新 List 和 Watch 的时候产生重复的记录。

Informer 组件不断地从 Delta 队列中弹出 delta 记录，然后把资源对象交给 indexer，让 indexer 把资源记录在一个缓存中，缓存在默认设置下是用资源的命名空间来做索引的，并且可以被 Controller Manager 或多个 Controller 所共享。之后，再把这个事件交给事件的回调函数

3. 控制循环例子 - 扩容
ReplicaSet 是一个用来描述无状态应用的扩缩容行为的资源， ReplicaSet controler 通过监听 ReplicaSet 资源来维持应用希望的状态数量，ReplicaSet 中通过 selector 来匹配所关联的 Pod，在这里考虑 ReplicaSet rsA 的，replicas 从 2 被改到 3 的场景。

首先，Reflector 会 watch 到 ReplicaSet 和 Pod 两种资源的变化，为什么我们还会 watch pod 资源的变化稍后会讲到。发现 ReplicaSet 发生变化后，在 delta 队列中塞入了对象是 rsA，而且类型是更新的记录。

Informer 一方面把新的 ReplicaSet 更新到缓存中，并与 Namespace nsA 作为索引。另外一方面，调用 Update 的回调函数，ReplicaSet 控制器发现 ReplicaSet 发生变化后会把字符串的 nsA/rsA 字符串塞入到工作队列中，工作队列后的一个 Worker 从工作队列中取到了 nsA/rsA 这个字符串的 key，并且从缓存中取到了最新的 ReplicaSet 数据。

Worker 通过比较 ReplicaSet 中 spec 和 status 里的数值，发现需要对这个 ReplicaSet 进行扩容，因此 ReplicaSet 的 Worker 创建了一个 Pod，这个 pod 中的 Ownereference 取向了 ReplicaSet rsA。

然后 Reflector Watch 到的 Pod 新增事件，在 delta 队列中额外加入了 Add 类型的 deta 记录，一方面把新的 Pod 记录通过 Indexer 存储到了缓存中，另一方面调用了 ReplicaSet 控制器的 Add 回调函数，Add 回调函数通过检查 pod ownerReferences 找到了对应的 ReplicaSet，并把包括 ReplicaSet 命名空间和字符串塞入到了工作队列中。

ReplicaSet 的 Woker 在得到新的工作项之后，从缓存中取到了新的 ReplicaSet 记录，并得到了其所有创建的 Pod，因为 ReplicaSet 的状态不是最新的，也就是所有创建 Pod 的数量不是最新的。因此在此时 ReplicaSet 更新 status 使得 spec 和 status 达成一致。

### 控制器模式总结
1. 两种 API 设计方法
Kubernetes 控制器模式依赖声明式的 API
2. 命令式 API 的问题
命令 API 最大的一个问题在于错误处理；
在大规模的分布式系统中，错误是无处不在的。一旦发出的命令没有响应，调用方只能通过反复重试的方式来试图恢复错误，然而盲目的重试可能会带来更大的问题。

假设原来的命令，后台实际上已经执行完成了，重试后又多执行了一个重试的命令操作。为了避免重试的问题，系统往往还需要在执行命令前，先记录一下需要执行的命令，并且在重启等场景下，重做待执行的命令，而且在执行的过程中，还需要考虑多个命令的先后顺序、覆盖关系等等一些复杂的逻辑情况。

实际上许多命令式的交互系统后台往往还会做一个巡检的系统，用来修正命令处理超时、重试等一些场景造成数据不一致的问题；

然而，因为巡检逻辑和日常操作逻辑是不一样的，往往在测试上覆盖不够，在错误处理上不够严谨，具有很大的操作风险，因此往往很多巡检系统都是人工来触发的。

最后，命令式 API 在处理多并发访问时，也很容易出现问题；
假如有多方并发的对一个资源请求进行操作，并且一旦其中有操作出现了错误，就需要重试。那么最后哪一个操作生效了，就很难确认，也无法保证。很多命令式系统往往在操作前会对系统进行加锁，从而保证整个系统最后生效行为的可预见性，但是加锁行为会降低整个系统的操作执行效率。

相对的，声明式 API 系统里天然地记录了系统现在和最终的状态。
不需要额外的操作数据。另外因为状态的幂等性，可以在任意时刻反复操作。在声明式系统运行的方式里，正常的操作实际上就是对资源状态的巡检，不需要额外开发巡检系统，系统的运行逻辑也能够在日常的运行中得到测试和锤炼，因此整个操作的稳定性能够得到保证。
3. 控制器模式总结
	* Kubernetes 所采用的控制器模式，是由声明式 API 驱动的。确切来说，是基于对Kubernetes 资源对象的修改来驱动的；
	* Kubernetes 资源之后，是关注该资源的控制器。这些控制器将异步的控制系统向设置的终态驱近；这些控制器是自主运行的，使得系统的自动化和无人值守成为可能；
	* 因为 Kubernetes 的控制器和资源都是可以自定义的，因此可以方便的扩展控制器模式。特别是对于有状态应用，我们往往通过自定义资源和控制器的方式，来自动化运维操作。这个也就是后续会介绍的 operator 的场景。

## 应用编排与管理

### Deployment: 管理部署发布的控制器
* deployment定义一种 Pod 期望数量，controller会持续维持 Pod 数量为期望的数量，当出现问题时候，controller会帮我们恢复，新扩出来对应的 Pod，保证可用的 Pod数量与期望数量一致
* 配置 Pod 发布方式。 controller 会按照用户给定的策略来更新 Pod，更新过程中，也可以设定不可用 Pod 数量在多少范围内。

### 用例解读
#### Deployment Status
* Processing
* Complete
* Failed
### 架构设计
#### 管理模式
* Deployment 只负责管理不同版本的Replicaset，由Replicaset管理Pod副本数。
* 每个ReplicaSet 对应了Deployment template 的一个版本
* 一个ReplicaSet 下的Pod都是相同的版本

#### Deployment 控制器
所有的控制器都是通过 Informer 中的 Event 做一些 Handler 和 Watch。这个地方 Deployment 控制器，其实是关注 Deployment 和 ReplicaSet 中的 event，收到事件后会加入到队列中。而 Deployment controller 从队列中取出来之后，它的逻辑会判断 Check Paused，这个 Paused 其实是 Deployment 是否需要新的发布，如果 Paused 设置为 true 的话，就表示这个 Deployment 只会做一个数量上的维持，不会做新的发布。

如果 Check paused 为 Yes 也就是 true 的话，那么只会做 Sync replicas。也就是说把 replicas sync 同步到对应的 ReplicaSet 中，最后再 Update Deployment status，那么 controller 这一次的 ReplicaSet 就结束了。

那么如果 paused 为 false 的话，它就会做 Rollout，也就是通过 Create 或者是 Rolling 的方式来做更新，更新的方式其实也是通过 Create/Update/Delete 这种 ReplicaSet 来做实现的。

#### ReplicaSet 控制器
当 Deployment 分配 ReplicaSet 之后，ReplicaSet 控制器本身也是从 Informer 中 watch 一些事件，这些事件包含了 ReplicaSet 和 Pod 的事件。从队列中取出之后，ReplicaSet controller 的逻辑很简单，就只管理副本数。也就是说如果 controller 发现 replicas 比 Pod 数量大的话，就会扩容，而如果发现实际数量超过期望数量的话，就会删除 Pod。

上面 Deployment 控制器的图中可以看到，Deployment 控制器其实做了更复杂的事情，包含了版本管理，而它把每一个版本下的数量维持工作交给 ReplicaSet 来做。


#### 扩/缩容模拟
下面来看一些操作模拟，比如说扩容模拟。这里有一个 Deployment，它的副本数是 2，对应的 ReplicaSet 有 Pod1 和 Pod2。这时如果我们修改 Deployment replicas， controller 就会把 replicas 同步到当前版本的 ReplicaSet 中，这个 ReplicaSet 发现当前有 2 个 Pod，不满足当前期望 3 个，就会创建一个新的 Pod3。


#### 发布模拟
我们再模拟一下发布，发布的情况会稍微复杂一点。这里可以看到 Deployment 当前初始的 template，比如说 template1 这个版本。template1 这个 ReplicaSet 对应的版本下有三个 Pod：Pod1，Pod2，Pod3。

这时修改 template 中一个容器的 image， Deployment controller 就会新建一个对应 template2 的 ReplicaSet。创建出来之后 ReplicaSet 会逐渐修改两个 ReplicaSet 的数量，比如它会逐渐增加 ReplicaSet2 中 replicas 的期望数量，而逐渐减少 ReplicaSet1 中的 Pod 数量。

那么最终达到的效果是：新版本的 Pod 为 Pod4、Pod5和Pod6，旧版本的 Pod 已经被删除了，这里就完成了一次发布。


#### 回滚模拟
来看一下回滚模拟，根据上面的发布模拟可以知道 Pod4、Pod5、Pod6 已经发布完成。这时发现当前的业务版本是有问题的，如果做回滚的话，不管是通过 rollout 命令还是通过回滚修改 template，它其实都是把 template 回滚为旧版本的 template1。

这个时候 Deployment 会重新修改 ReplicaSet1 中 Pod 的期望数量，把期望数量修改为 3 个，且会逐渐减少新版本也就是 ReplicaSet2 中的 replica 数量，最终的效果就是把 Pod 从旧版本重新创建出来。

#### spec 字段解析
最后再来简单看一些 Deployment 中的字段解析。首先看一下 Deployment 中其他的 spec 字段：

* MinReadySeconds：Deployment 会根据 Pod ready 来看 Pod 是否可用，但是如果我们设置了 MinReadySeconds 之后，比如设置为 30 秒，那 Deployment 就一定会等到 Pod ready 超过 30 秒之后才认为 Pod 是 available 的。Pod available 的前提条件是 Pod ready，但是 ready 的 Pod 不一定是 available 的，它一定要超过 MinReadySeconds 之后，才会判断为 available；
* revisionHistoryLimit：保留历史 revision，即保留历史 ReplicaSet 的数量，默认值为 10 个。这里可以设置为一个或两个，如果回滚可能性比较大的话，可以设置数量超过 10；
* paused：paused 是标识，Deployment 只做数量维持，不做新的发布，这里在 Debug 场景可能会用到；
* progressDeadlineSeconds：前面提到当 Deployment 处于扩容或者发布状态时，它的 condition 会处于一个 processing 的状态，processing 可以设置一个超时时间。如果超过超时时间还处于 processing，那么 controller 将认为这个 Pod 会进入 failed 的状态。

#### 升级策略字段解析
Deployment 在 RollingUpdate 中主要提供了两个策略，一个是 MaxUnavailable，另一个是 MaxSurge。这两个字段解析的意思，可以看下图中详细的 comment，或者简单解释一下：

* MaxUnavailable：滚动过程中最多有多少个 Pod 不可用；
* MaxSurge：滚动过程中最多存在多少个 Pod 超过预期 replicas 数量。
上文提到，ReplicaSet 为 3 的 Deployment 在发布的时候可能存在一种情况：新版本的 ReplicaSet 和旧版本的 ReplicaSet 都可能有两个 replicas，加在一起就是 4 个，超过了我们期望的数量三个。这是因为我们默认的 MaxUnavailable 和 MaxSurge 都是 25%，默认 Deployment 在发布的过程中，可能有 25% 的 replica 是不可用的，也可能超过 replica 数量 25% 是可用的，最高可以达到 125% 的 replica 数量。
这里其实可以根据用户实际场景来做设置。比如当用户的资源足够，且更注重发布过程中的可用性，可设置 MaxUnavailable 较小、MaxSurge 较大。但如果用户的资源比较紧张，可以设置 MaxSurge 较小，甚至设置为 0，这里要注意的是 MaxSurge 和 MaxUnavailable 不能同时为 0。

理由不难理解，当 MaxSurge 为 0 的时候，必须要删除 Pod，才能扩容 Pod；如果不删除 Pod 是不能新扩 Pod 的，因为新扩出来的话，总共的 Pod 数量就会超过期望数量。而两者同时为 0 的话，MaxSurge 保证不能新扩 Pod，而 MaxUnavailable 不能保证 ReplicaSet 中有 Pod 是 available 的，这样就会产生问题。所以说这两个值不能同时为 0。用户可以根据自己的实际场景来设置对应的、合适的值。

### Summary
* Deployment 是 Kubernetes 中常见的一种 Workload，支持部署管理多版本的 Pod；
* Deployment 管理多版本的方式，是针对每个版本的 template 创建一个 ReplicaSet，由 * ReplicaSet 维护一定数量的 Pod 副本，而 Deployment 只需要关心不同版本的 ReplicaSet 里要指定多少数量的 Pod；
* Deployment 发布部署的根本原理，就是 Deployment 调整不同版本 ReplicaSet 里的终态副本数，以此来达到多版本 Pod 的升级和回滚。

### Job
 * kubernetes 的 Job 是一个管理任务的控制器，它可以创建一个或多个 Pod 来指定 Pod 的数量，并可以监控它是否成功地运行或终止；
* 可以根据 Pod 的状态来给 Job 设置重置的方式及重试的次数；
* 还可以根据依赖关系，保证上一个任务运行完成之后再运行下一个任务；
* 同时还可以控制任务的并行度，根据并行度来确保 Pod 运行过程中的并行次数和总体完成大小。
#### 并行运行Job

有时候有些需求：希望 Job 运行的时候可以最大化的并行，并行出 n 个 Pod 去快速地执行。同时，由于我们的节点数有限制，可能也不希望同时并行的 Pod 数过多，有那么一个管道的概念，我们可以希望最大的并行度是多少，Job 控制器都可以帮我们来做到。

这里主要看两个参数: 一个是 completions，一个是 parallelism。

* completions: 是用来指定本 Pod 队列执行次数。可能这个不是很好理解，其实可以把它认为是这个 Job 指定的可以运行的总次数。比如这里设置成 8，即这个任务一共会被执行 8 次；
* parallelism: 代表这个并行执行的个数。所谓并行执行的次数，其实就是一个管道或者缓冲器中缓冲队列的大小，把它设置成 2，也就是说这个 Job 一定要执行 8 次，每次并行 2 个 Pod，这样的话，一共会执行 4 个批次。

### CronJob

### 架构设计
#### Job管理模式
1. Job Controller 负责根据配置创建Pod。
2. Job Controller 跟踪Job状态，根据配置及时重试或者继续创建。
3. Job Controller 会自动添加label 来跟踪对应的pod，并根据配置并行或者串行创建pod。

### DaemonSet 守护进程控制器

DaemonSet 也是 Kubernetes 提供的一个 default controller，它实际是做一个守护进程的控制器，它能帮我们做到以下几件事情：

* 首先能保证集群内的每一个节点都运行一组相同的 pod；
* 同时还能根据节点的状态保证新加入的节点自动创建对应的 pod；
* 在移除节点的时候，能删除对应的 pod；
* 而且它会跟踪每个 pod 的状态，当这个 pod 出现异常、Crash 掉了，会及时地去 recovery 这个状态。
#### DaemonSet语法
适用场景
1. 集群存储进程: glusterd、ceph
2. 日志收集进程: fluentd、logstash
3. 需要在每个节点运行的监控收集器

#### 更新 DaemonSet

DaemonSet 和 deployment 特别像，它也有两种更新策略：一个是 RollingUpdate，另一个是 OnDelete。

* RollingUpdate 其实比较好理解，就是会一个一个的更新。先更新第一个 pod，然后老的 pod 被移除，通过健康检查之后再去见第二个 pod，这样对于业务上来说会比较平滑地升级，不会中断；
* OnDelete 其实也是一个很好的更新策略，就是模板更新之后，pod 不会有任何变化，需要我们手动控制。我们去删除某一个节点对应的 pod，它就会重建，不删除的话它就不会重建，这样的话对于一些我们需要手动控制的特殊需求也会有特别好的作用。

#### DaemonSet管理模式
1. DaemonSet Controller 负责根据配置创建Pod
2. DaemonSet Controller 跟踪 Job 状态，根据配置及时重试Pod或者继续创建
3. DaemonSet Controller 会自动添加affinity & label来跟踪对应的Pod，并根据配置在每个节点或者适合的部分节点创建Pod
#### DaemonSet 的控制器

DaemonSet 其实和 Job controller 做的差不多：两者都需要根据 watch 这个 API Server 的状态。现在 DaemonSet 和 Job controller 唯一的不同点在于，DaemonsetSet Controller需要去 watch node 的状态，但其实这个 node 的状态还是通过 API Server 传递到 ETCD 上。

当有 node 状态节点发生变化时，它会通过一个内存消息队列发进来，然后DaemonSet controller 会去 watch 这个状态，看一下各个节点上是都有对应的 Pod，如果没有的话就去创建。当然它会去做一个对比，如果有的话，它会比较一下版本，然后加上刚才提到的是否去做 RollingUpdate？如果没有的话就会重新创建，Ondelete 删除 pod 的时候也会去做 check 它做一遍检查，是否去更新，或者去创建对应的 pod。

当然最后的时候，如果全部更新完了之后，它会把整个 DaemonSet 的状态去更新到 API Server 上，完成最后全部的更新。