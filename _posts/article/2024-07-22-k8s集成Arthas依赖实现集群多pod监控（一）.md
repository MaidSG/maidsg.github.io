---

layout: post

title: "2024-07-22-k8s集成Arthas依赖实现集群多pod监控(一)"

date:   2024-07-22

tags: [thinking,心理学,有效努力,basic knowledge]

comments: true

author: 阿昱

---


## 背景介绍
---
	开始之前先简单记录一下自己在整个过程中了解和学习到的知识，当我们一腔热血开始尝试之前，我们一定不能陷入进去，应该时刻提醒自己，不要只顾眼前的任务和急于求成，而是应该怀着体验技术，接触新思想和事物的心态去专注完成，完成后或在这个过程中记录下自己学的东西，才能完成一次完美学习的闭环，突破自己，获取更好的技术提升。
### Arthas
Arthas 是一个开源的 Java 诊断工具，由阿里巴巴开源。它主要用于在生产环境中对 Java 应用进行监控、诊断。Arthas 允许开发者在不停止应用的情况下，动态地查看应用的运行状态，包括但不限于：
- **JVM 实时监控**：查看 JVM 的实时状态，包括线程堆栈、内存使用、GC 状态等。
- **Class 和 Method 诊断**：可以查看和监控 Java 类的加载信息，方法的调用统计，甚至可以在运行时修改类的定义。
- **SQL 诊断**：对数据库操作进行监控，查看 SQL 执行时间，帮助定位性能瓶颈。
- **系统指标监控**：查看系统的各种指标，如 CPU、内存、磁盘、网络等。
- **命令行交互**：提供了一个命令行工具，通过各种命令来实现对应用的诊断。
### Arthas Spring Boot Starter
Arthas Spring Boot Starter 是一个为 Spring Boot 应用集成阿里巴巴开源 Java 诊断工具 Arthas 的启动器，当springboot集成了Starter后，无需重启应用，即可动态地连接到应用进程，执行各种诊断命令。Arthas agent会跟随项目一起启动，应用启动后，spring 会启动 arthas，并且 attach 自身进程。集成后springboot服务可以通过配置 tunnel server 实现远程管理。

### # Arthas Tunnel
Arthas Tunnel 的工作原理是在目标 Java 应用上启动一个 Arthas Agent，这个 Agent 会创建一个到 Arthas Tunnel Server 的连接。用户可以通过访问 Arthas Tunnel Server 提供的 Web 控制台，来远程执行 Arthas 支持的各种诊断命令，就像直接在目标机器上操作一样。Arthas Tunnel 与被监听对象（即目标 Java 应用）建立的连接是一个基于 WebSocket 或 HTTP 的长连接。这种连接允许双向通信，即 Arthas Tunnel Server 可以接收来自被监听对象的数据，同时也可以发送命令到被监听对象。

### Kubernetes
Kubernetes（通常简称为 k8s）是一个开源平台，用于自动部署、扩展和管理容器化应用程序。在 Kubernetes 中，Pod 是最小的部署单元，可以被创建和管理。

一个 Pod 通常包含一个或多个容器（例如 Docker 容器）。如果一个 Pod 包含多个容器，这些容器会被部署在同一个宿主机上，并且可以共享资源。每个 Pod 都有一个独特的 IP 地址，允许容器使用网络进行通信。

Pod 的关键特性包括：
- **共享网络**：Pod 内的所有容器共享一个 IP 地址和端口空间，它们可以通过 localhost 相互通信。
- **共享存储**：Pod 可以定义共享存储卷，这些存储卷可以被 Pod 内的所有容器访问。
- **生命周期**：通常，Pod 中的容器会一起启动、一起停止。Pod 作为一个整体被调度到节点上，它们共享相同的生命周期、调度上下文和系统资源。
- **自我治愈**：Kubernetes 会自动替换发生故障的 Pod，确保所部署的应用数量符合用户的期望状态。

Pods 是 Kubernetes 应用的基石，它们使得容器化应用的部署和管理变得更加灵活和高效。
设计Pod间通信时，推荐使用Service，- Kubernetes 的 Service 是一种抽象，它定义了一种访问 Pod 的方式，无论背后有多少 Pod 实例。Service 为一组具有相同功能的 Pod 提供一个单一的入口地址（IP 地址和端口号），其他 Pod 可以通过这个地址访问后端的一组 Pod。Kubernetes (k8s) 集群中的 Pods 互相通信和访问。

## 部署准备
---
**个人实践经验：**
- k8s集群：阿里云容器服务 Kubernetes 版（ACK）
- 目标服务jdk版本： 1.8 或者jdk 21
- springboot版本： springboot 2.x 或者springboot3.x
- Arthas源码(主要用于定制arthas-tunnel-server 2024-07-18 v3.7.2)：https://github.com/alibaba/arthas.git
- arthas-spring-boot-starter v3.7.2： [Maven Central Repository Search](https://search.maven.org/search?q=arthas-spring-boot-starter)
- 阿里云docker镜像仓库(私服仓库，用于本地打包镜像部署到集群)
- 注意⚠️⚠️：java服务一定要使用jdk环境，单纯jre环境无法使用arthas，例如Arthas 有时需要编译用户输入的 Java 代码片段这时就需要通过编译器工具javac才能实现；Arthas 需要使用到 JDK 中提供的高级类加载机制，以便它能够在运行时动态加载、更新和卸载类。所以一定要检查部署的java服务使用的镜像是否用jdk，官方建议使用`openjdk:8-jdk`或者`openjdk:8-jdk-alpine`。
## 实践环节
---
### 目标springboot应用集成Starter，配置信道
- 引入starter依赖
```xml
<dependency>  
    <groupId>com.taobao.arthas</groupId>  
    <artifactId>arthas-spring-boot-starter</artifactId>  
    <version>3.7.2</version>  
</dependency>

```
- 配置项添加server信道，详细配置可参考官网文档[Arthas Properties | arthas (aliyun.com)](https://arthas.aliyun.com/doc/arthas-properties.html)
```properties
arthas.appName=demoapp
arthas.tunnelServer=ws://127.0.0.1:7777/ws
arthas.agentId=mmmmmmyiddddd
```
添加好后启动项目，如果配置正常可以在控制台看到Arthas自动装配配置的日志输出：
![https://s2.loli.net/2024/07/18/o9Ck7bFARNJmP2X.png](https://s2.loli.net/2024/07/18/o9Ck7bFARNJmP2X.png)

如果是springboot3 目前的3.7.2也是支持的，需要注意的是需要自己按照springboot3的特性将配置文件import参考：[Arthas Spring Boot Starter 有计划支持 SpringBoot3 吗？ · Issue #2599 · alibaba/arthas (github.com)](https://github.com/alibaba/arthas/issues/2599)
[arthas-spring-boot-starter 需要支持 spring boot3 · Issue #2524 · alibaba/arthas (github.com)](https://github.com/alibaba/arthas/issues/2524)
需要在META-INF.spring（名称如此）文件夹下创建如下文件：org.springframework.boot.autoconfigure.AutoConfiguration.imports
添加配置项：
```java
# 以下是Arthas的自动装配配置文件 4.0发布后不再需要  
com.alibaba.arthas.spring.ArthasConfiguration  
com.alibaba.arthas.spring.endpoints.ArthasEndPointAutoConfiguration
```

### 打包部署Tune Server 
- 首先先拉一下官方的源码，用于本地打包
```shell
https://github.com/alibaba/arthas.git
```
拉下来后，不要急着打包，官方提供了不同环境的打包配置，默认应该是会自动检测系统进行匹配，确实牛的，非常方便。由于我的是macOS，需要修改一下配置maven打包时才能正常检测到java的path如果你打包的环境也是的话，需要记得修改一下arthas-vmtool的pom.xml
将pom.xml中的`${JAVA_HOME}`替换为 `${env.JAVA_HOME}`。
![image.png](https://s2.loli.net/2024/07/18/2PCsyQoYGKM3ca6.png)

- 定制一下server的配置或功能，我主要将端口号和密码改了，server使用的是Spring Scurity，修改密码只用改配置文件就可以了。
- 执行maven命令进行打包,这是我的打包命令：
```shell
mvn clean package -Dmaven.test.skip=true
```
- 编写构建镜像的Dockerfile，和tunnel-server的pom.xml放一起。这里我使用的基础镜像是公司的私服镜像
```shell
FROM registry.cn-beijing.aliyuncs.com/devops-op/openjdk:1.8.0_212  
MAINTAINER bixiaoyu@bjca.org.cn  
  
#ENV TZ=Asia/Shanghai  
RUN mkdir /data/webserver -p  
  
ENV LANG en_US.UTF-8  
ADD start.sh /data/webserver/  
RUN chmod +x /data/webserver/start.sh  
  
#COPY agent /data/agent  
ADD ./target/arthas-tunnel-server-4.0.0-SNAPSHOT-sources.jar   /data/webserver/  
#RUN apk add --update ttf-dejavu fontconfig  
  
EXPOSE 7777  
EXPOSE 9342  
  
ENTRYPOINT ["/data/webserver/start.sh"]
```
- 构建镜像并且上传到公司私服，后续方便部署到阿里云k8s
```shell
#1 构建业务镜像
sudo docker build -t maidsg/arthas-tunnel-server:1.0 . 

#2 调整镜像名，匹配私服
docker tag maidsg/arthas-tunnel-server:1.0  [registry.cn-beijing.aliyuncs.com/devops-op/arthas-tunnel-server:1.0](http://registry.cn-beijing.aliyuncs.com/devops-op/arthas-tunnel-server:1.0)

#3 上传私服
 docker push registry.cn-beijing.aliyuncs.com/devops-op/arthas-tunnel-server:1.0
```
其实镜像构建好后就可以先在本地用docker跑一下看看效果了，也可以连上目标服务写一个docker-compose先在本地验证一下监控的效果。

### 集群环境部署验证
- 部署server
server记得开放好端口，同时注意网络服务名称千万不要设置为arthas-server，因为这样启动后，给server配置的系统环境变量端口会和应用里的arthas.server.port 冲突。
![image.png](https://s2.loli.net/2024/07/20/JnBemolKAX8EfzQ.png)
部署好后，配置网管路由后就能在公网查看了。
![image.png](https://s2.loli.net/2024/07/20/yF5TPlM8URB67XS.png)
![image.png](https://s2.loli.net/2024/07/20/u35anbxTh9DEmS2.png)

- 部署目标服务
![image.png](https://s2.loli.net/2024/07/20/zEWAxPNrVktwmvb.png)

完成以上操作后，你的集群有了一个用于监控的pod，arthas server，然后后续需要监控的目标服务加上starter依赖就可以通过arthas server进行访问，但是注意目前还并没用完成完整实现监控方案。
虽然能在server的web console，或server对外的url/actuator/arthas中看到具体的连接信息，但是你使用控制台会发现仍然无法访问，这是因为浏览器访问时，当前浏览器与目标服务的 agent的网络不通而无法访问arthas agent的问题，因为线上的网络和k8s中server、agent配置的参数都是k8s集群的网络策略；
目前比较有效的解决方案就是通过给tunnel server服务建立websocket代理，浏览器的websocket请求在server服务中转发，方案我参考这位大佬实现的：[Arthas Tunnel Server功能扩展 - 掘金 (juejin.cn)](https://juejin.cn/post/7003001038181498910)
后续的实践流程我放到了k8s集成Arthas依赖实现集群多pod监控（二）中，记录不易，如果有帮到你，请给我点个赞🙏