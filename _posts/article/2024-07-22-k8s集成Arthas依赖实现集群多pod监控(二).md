---

layout: post

title: "2024-07-22-k8s集成Arthas依赖实现集群多pod监控(二)"

date:   2024-07-22

tags: [k8s,服务器,arthas,basic knowledge]

comments: true

author: 阿昱

---



## 概念概览
---
### Spring WebFlux
webflux 采用异步非阻塞 I/O 操作，基于 Reactor 模式，支持反应式编程，相比Spring MVC基于Servlet API 和阻塞 I/O 操作，webflux 在Netty上运行，借助Netty框架的网络编程抽象和工具类，webflux 允许少量线程处理大量并发请求，通过发布者-订阅者模式和回调方式来处理数据流和异步执行。
因此WebFlux 适合需要处理大量并发请求的场景，如实时数据处理、高性能 API 网关等。对于 I/O 密集型任务，如远程服务调用、数据库操作，可以更有效地利用系统资源。
对于在k8s上的Arthas Tunnel Server/Client监控方案来说，由于浏览器并不能直接通过公网与集群中的服务建立websocket连接，目前可行的一种方案，就是需要Arthas Tunnel Server服务能够代理浏览器进行与目标监控服务进行websocket连接，浏览器将websocket请求提交到tunnel server中，由tunnel server中做转发代理。

### Spring Security WebFlux
Spring Security WebFlux 是 Spring Security 框架为响应式编程模型提供的安全解决方案，当spring webflux应用集成后，可以使服务提供与 Spring Security 相同级别的安全保护，同时保持高性能和响应式编程的优势。
相较于支持Spring MVC的Spring Security，对于WebFlux的支持，Spring Security的配置文件的注解使用的是`@EnableWebFluxSecurity`；此外，Spring Security通过 `SecurityWebFilterChain` 定义WebFlux服务的HTTP请求的安全规则。

## 部署准备
---
- 前期集群搭建，服务连通以及配置文件的选定
- 根据Arthas官方源码，改造Arthas Tunnel Server，替换Restful MVC为Spring WebFlux，或者直接使用大佬的拓展项目：[Arthas Tunnel Server功能扩展 - 掘金 (juejin.cn)](https://juejin.cn/post/7003001038181498910)进行改造（推荐）

## 实践环节
### 改造Arthas Tunnel Server 替换为WebFlux
大佬的拓展项目中，Tunnel Server的访问路径为"/",如果集群使用时，推荐使用二级域名进行ngix网络转发配置，我使用的是添加路由前缀的方案因此需要修改项目中涉及到相关路径的地方。
![image.png](https://s2.loli.net/2024/07/22/NbRuU4PTXy1qOAY.png)
如果你也选择添加路由前缀的方案，记得调整Spring Security初始默认的表单登录，需要添加html，并且配置相关的跳转路径。
![image.png](https://s2.loli.net/2024/07/22/POzDkf3lBFX8LYg.png)
默认的表单logout只支持post请求的访问，也就是在登出的时候区分是`接口登出`还是`页面登出`默认springsecurit使用的是接口发送POST请求进行登出，但是参考官网文档[LogoutConfigurer (spring-security-docs 6.3.1 API)](https://docs.spring.io/spring-security/site/docs/current/api/org/springframework/security/config/annotation/web/configurers/LogoutConfigurer.html#logoutSuccessUrl-java.lang.String-)可以通过给`logoutUrl`设置一个Get请求的url进行登出处理。对于WebFlux，使用`.requiresLogout()`配合`ServerWebExchangeMatchers.pathMatchers`可以实现：
![image.png](https://s2.loli.net/2024/07/22/qHpsvPc7LVEl6gT.png)
### 打包部署改造后的Arthas Tunnel Server
- 打包
```shell
mvn clean package -Dmaven.test.skip=true
```
- 部署，私服的做法，需要提前登录私服
```shell
# 构建镜像
 sudo docker build -t 【镜像名】 .

# 调整镜像名
docker tag【镜像名】【仓库访问镜像名】

# 上传镜像
docker push 【仓库访问镜像名】

```
部署后配置好网络路由，就可以使用了。
![image.png](https://s2.loli.net/2024/07/22/eu2ozqk7OrYBgbZ.png)
