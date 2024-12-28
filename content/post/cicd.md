---
categories:
- devops
date: 2024-03-09 22:01:35
categories:
- cicd
tags:
- cicd
---

在 DevOps 领域中，我们通常需要实现cicd，持续地支撑业务团队软件交付，在云原生时代，cicd通常是如何实现的呢？

### CI（持续集成）

在云原生时代，持续集成包括对源代码的质量检查，代码扫描，安全漏洞治理等，然后通过镜像构建，交付出一个 应用的制品包，即docker 镜像。

常见的应用持续集成工具有：

* Jenkins
  * 优点：完全自定义，可以自己定义流水线，定义每个执行步骤，拉取代码，构建代码，镜像打包上传harbour
* Gitlab ci
  * 需要管理每个git仓库的.gitlab-ci.yml文件
* github actions

### CD（持续交付）

将镜像作为容器部署起来



