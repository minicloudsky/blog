---
title: 云原生课程学习笔记
date: 2021-01-15 14:01:35
tags:
---


## 云原生技术范畴
- 不可变基础设施 目前实现: 容器镜像
- 云应用编排理论 目前实现: 容器设计模式
![](cncf.png)

### 什么是容器？
容器，是一个视图隔离，资源可限制，独立文件系统的进程集合
- 视图隔离 - 如能看到部分进程，独立主机名等等；
- 控制资源使用率 - 如 2G 内存大小，CPU 使用个数等等
<!-- more -->
### 什么是镜像？
运行容器所需要的所有文件集合 - 容器镜像
Dockerfile - 描述镜像构建步骤
构建步骤所产生出文件系统的变化- changeset
类似 disk snapshot
提高分发效率，减少磁盘压力

### summary
容器 - 和系统其它部分隔离开的进程集合
镜像 - 容器所需要的所有文件集合 - Build once，Run anywhere

### 容器运行时的生命周期
- 单进程模型
   1. Init 进程生命周期 = 容器生命周期
   2. 运行期间可运行 exec 执行运维操作

- 数据持久化
   1. 独立于容器的生命周期
   2. 数据卷 - docker volume vs bind

### moby 容器引擎架构
- containerd 
   1. 容器运行时管理引擎，独立于 moby daemon
   2. containerd-shim 管理容器生命周期，可被 containerd 动态接管

- 容器运行时
   1. 容器徐计划技术方案
   2. runC kata gVisor


   moby daemon
   containerd
shim shim shim
container   container  container
runC         runC          runC

### 容器和VM之间的差异
![](compare.png)

### Kubernetes
自动化的容器编排平台
- 部署
- 弹性
- 管理
核心功能
- 服务发现与负载均衡
- 容器自动装箱
- 存储编排
- 自动容器恢复
- 自动发布与回滚
- 配置与密文管理
- 批量执行
- 水平伸缩

![](recover.png)
![](water.png)
![](schedluer.png)
### Kubernetes 架构
![](master.png)
![](node.png)
![](example.png)

### Kubernetes 核心概念与API
- Pod
   - 最小的调度以及资源单元
   - 由一个或多个容器组成
   - 定义容器运行的方式(Command 或者环境变量等)
   - 提供给容器共享的运行环境(网络、进程空间)
- Volume
   - 声明 Pod 中的容器可访问的文件目录
   - 可以被挂载在 Pod 中一个(或者多个) 容器的指定路径下
   - 支持多种后端存储的抽象
      - 本地存储、分布式存储、云存储。。
- Deployment
   - 定义一组 Pod 的副本数目、版本等
   - 通过控制器(COntroller) 维持Pod的数目
      - 自动恢复失败的 Pod
   - 通过控制器以指定的策略控制版本
      - 滚动升级、重新生成、回滚等
- Service
   - 提供访问一个或多个 Pod 实例的稳定访问地址
   - 支持多种访问方式实现
      - ClusterIP
      - NodePort
      - LoadBalancer
- Namespaces
   - 一个集群内部的逻辑隔离机制(鉴权，资源额度)
   - 每个资源都属于一个Namspace
   - 同一个Namspace 中的资源命名
   - 不同Namspace 中的资源可命名

### API 基础知识
HTTP + JSON/YAML
- kubectl
- UI
- curl

![](api.png)
![](label.png)
### 为何需要 Pod
#### 容器的本质
- 一个视图被隔离、资源受限的进程
- 容器里PID = 1 的进程就是应用本身
- 管理虚拟机 = 管理基础设施;管理容器 - 直接管理应用本身
Kubernetes 是云时代的的操作系统，以此类推，容器镜像其实就是: 这个操作系统的软件安装包

- Kubernetes = 操作系统(例如: Linux) 
- 容器= 进程(Linux 线程)
- Pod = 进程组(Linux 线程组)
### 进程组
helloworld程序由4个进程组成，这些进程之间共享某些文件
helloworld程序如何用容器跑起来？
解法1： 在一个Docker容器中，启动这4个进程
疑问： 容器 PID=1 的进程就是应用本身，蔽日 main 进程，那么谁来负责管理剩余的3个进程？
容器是单进程模型，除非，应用进程本身具备"进程管理"能力(这意味着: helloworld 程序需要具备 systemd 的能力)，或者，容器的 PID=1 进程改成 systemd，这会导致: `管理容器 = 管理 systemd != 直接管理应用本身`
Pod = 进程组
Pod 是一个逻辑单元，多个容器的组合，Kubernetes 的原子调度单位。
Google 工程师发现，在 Borg 项目部署的应用，往往都存在着类似于"进程和进程组"的关系，更具体说，就是这些应用之间有着密切的协作关系，使得它们必须部署在同一台机器上并且共享这些信息。

#### 为什么 Pod 必须是原子调度单位？
example: 容器间协作
- App： 业务容器，写日志文件
- LogCollector: 转发日志文件到Elasticsearch中
- 内存要求:
  - App: 1G
  - LogCollector: 0.5G
- 当前可用内存:
  - Node_A: 1.25G
  - Node_B: 2G
此时如果 App 先被调度到了Node_A上，会怎么样？
Node_A资源不足，App 将无法运行
#### Task Scheduling 问题
- Mesos: 资源囤积(resource hoarding)
  - 所有设置了Affinity 约束的任务都到达时，才开始统一进行调度
  - 调度效率损失和死锁
- Google Omega: 乐观调度处理冲突
  - 先不管冲突，通过回滚机制在出现冲突之后解决问题
  - 复杂
- Kubernetes: Pod
#### 亲密关系 - 调度解决
- 两个应用需要运行在同一个宿主机上
#### 超亲密关系 - Pod 解决
- 会发生直接的文件交换
- 使用 localhost 或者 Socket 文件进行本地通信
- 会发生非常频繁的RPC调用
- 会共享某些 Linux Namespace(比如，一个容器要加入另一个容器额Network Namespace)
### Pod 要解决的问题
如何让一个 Pod 里多个容器之间最高效的共享某些资源和数据？
容器之间原本是被 Linux Namespace 和 cgroups 隔离开
#### 共享网络
容器 A 和 B
- 通过 Infra Contanier 方式共享同一个Network Namespace
  - 镜像: k8s.gcr.io/pause;汇编语言便携的，永远处于暂停，大小 100-200kb
  - 直接使用 localhost 通信
  - 看到的网络设备和Infra容器看到的一样
  - 一个 Pod 只有一个 ip 地址，也就是这个 Pod 的Network Namespace 对应的 ip 地址
    - 所有网络资源，都是一个 Pod 一份，并且被该 Pod 中的所有容器共享
  - 整个 Pod 的生命周期和 Infra 容器一一致，而与容器 A 和 B 无关

### 共享存储
![](storage.png)
### 容器设计模式
war 包 + Tomcat 容器化
- 方法-: 把war包和Tomcat打包进一个镜像
  - 无论是war包还是tomcat更新都需要重新制作镜像
- 方法二: 镜像只打包 tomcat，使用数据卷(hostpath)从宿主机上将 war 包挂载进 tomcat 容器
  - 需要维护一套分布式存储系统
### InitContainer
![](initcontainer.png)
### 容器设计模式： Sidebar
通过在 Pod 里定义专门容器，来执行主业务容器的辅助工作
- 原本需要ssh进行执行的脚本
- 日志收集
- debug应用
- 应用监控
优势在于，将辅助功能和主业务容器解耦，实现独立发布和能力重用
### Sidebar: 应用于日志收集
![](log.png)
### Sidebar: 代理容器
![](proxy.png)
### Sidebar: 适配器容器
![](shipeiqi.png)
## 应用编排与管理: 核心原理
### Kubernetes 资源对象
- Spec: 期望的状态
- Status: 观测到的状态
- Metadata:
  - Labels
  - Annotations
  - OwnerReference
#### Labels
- 标识型的Key:Value 元数据
- 作用
  - 用于筛选资源
  - 唯一的组合资源的方法
- 可以使用 Selector 来查询
  - 类似于 SQL 'select * where ...'
例子:
```yaml
environment: production
release: stable
app.kubernetes.io/version: 5.7.21
failure-domain.beta.kubernetes.io/region: cn-hangzhou
```
### Selector
![](selector.png)
![](selector2.png)
![](selector3.png)
### annotations
![](annotations.png)
### Ownereference
![](owner.png)
### 控制循环
![](control.png)
### sensor
![](sensor.png)
### controller
![](ctrl.png)
### 控制循环例子 - 扩容
![](ctrl-example.png)
![](process.png)
![](process2.png)
### 两种 API 设计方法
- 命令式(和孩子交流)
  - 吃饭
  - 刷牙
  - 睡觉
  - 唱一首歌
  - 新扩一个 pod
  - 删除一个 pod
0 - 声明式(和员工交流)
    - 市场占有率达到 80%
    - 稳定性达到 99.99%
    - 做一个身高体重正常的孩子
    - 副本数保持在 3 个
### 命令式和声明式对比
#### 命令式
- 命令未响应怎么办?
  - 反复重试
  - 需要记录当前的操作 复杂
- 如果多重试了怎么办？
  - 巡检做修正 - 额外工作，危险
- 如果多方并发访问怎么办
  - 需要加锁，复杂低效
#### 声明式
- 天然地记录了状态
- 幂等操作，可在任意时刻反复操作
- 正常操作即巡检
- 可合并多个变更
### 控制器模式总结
![](ctrl4.png)
