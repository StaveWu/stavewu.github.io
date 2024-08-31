---
title: kubernetes
date: 2024/03/05 23:00:00
toc: true
categories: 
 - 云组件
tags: 
 - 虚拟化
---

本文尝试部署kubernetes集群，并理解该组件能够做什么，什么时候使用。

<!-- more -->

## 部署集群

假设我们有3台设备，每台设备都已修改过hostname：

```bash
hostnamectl set-hostname xxxx
```

并已同步在/etc/hosts里配置了对应的ip hostname映射。

master、node01、node02

前置准备（此步骤在所有节点上做）：

```bash
# This overwrites any existing configuration in /etc/yum.repos.d/kubernetes.repo
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.29/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
yum install -y docker
```

环境准备，包括关闭selinux、关闭swap、关闭防火墙、打开IP forward、修改docker的cgroupfs驱动为systemd、准备docker-shim（此步骤在所有节点上做）：

```bash
setenforce 0
swapoff -a
systemctl stop firewalld
systemctl disable firewalld

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system

cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.10/cri-dockerd-0.3.10.arm64.tgz
tar -zxf cri-dockerd-0.3.10.arm64.tgz
cd cri-dockerd
install cri-dockerd /usr/bin

git clone https://github.com/Mirantis/cri-dockerd.git
cd cri-dockerd
install packaging/systemd/* /etc/systemd/system
systemctl daemon-reload
systemctl restart docker
systemctl start cri-docker.socket

# 启动集群服务进程
systemctl enable --now kubelet
```

我们现在在master节点上启动集群：

```bash
kubeadm init --apiserver-advertise-address=[网卡ip] --pod-network-cidr=10.224.0.0/16
```

配置.kube/config，否则无法查询集群信息：

```bash
mkdir ~/.kube
cp /etc/kubernetes/admin.conf ~/.kube/config
kubectl get pods --all-namespaces
```

此时有两个coredns pod应该是pending的，因为它在等待cni插件。

安装cni插件，这里选calico：

> 注意这里calico在arm上有bug，遵循[官网的yml](https://docs.tigera.io/calico/latest/getting-started/kubernetes/quickstart)会安装失败报libpcap.so找不到，跟calico版本有关，原因[见该issue](https://github.com/bettercap/bettercap/issues/98)

```bash
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml

wget https://docs.projectcalico.org/manifests/custom-resources.yaml
vim custom-resources.yaml
# 修改cidr为10.244.0.0/16
kubectl create -f custom-resources.yaml
```

安装成功后观察coredns是否为running。

如果coredns出现错误，通过kubectl describe coredns-xxxx -n kube-system查到bind 53: permission denied错误，则需要做如下变更：

```bash
kubectl edit cm coredns -n kube-system
# 修改为.:1053 {
#   forward . 8.8.8.8:53
#   ...
```

意思是监听1053，再转发到8.8.8.8:53地址。

修改后重启coredns：

```bash
kubectl delete pod coredns-xxxx -n kube-system
kubectl delete pod coredns-xxxx -n kube-system
```

重启后应能正确运行。

剩下的就是join工作节点了：

```bash
# 在master节点上运行
kubeadm token create --print-join-command
# 将输出的命令在其他节点上执行
kubeadm join 170.70.226.121:6443 --token jalb9p.6os29537lon9befl --discovery-token-ca-cert-hash sha256:923a907a5fb070aad4d91f4e5ce6a5b8c55b8c1d6d6e326eadc8c95eaa8987d2
# 成功后查询
kubectl get nodes
```

应全部ready。

至此集群搭建完成。

## 使用集群

准备一个样例：

数据库部分不在kubernetes上部署，只准备服务部分。

服务采用spring cloud，分几段：

*   服务网关spring cloud gateway
*   具体处理业务的服务

服务网关将通过rpc调用具体处理业务的服务。

网关和具体服务分开为两个pod

为了接近生产配置，网关设置出2个副本（分散在2个node），具体服务也同样处理。

**Gateway**

gateway作用：

![Spring Cloud Gateway diagram](https://static.spring.io/blog/fombico/20220826/spring-cloud-gateway-diagram.png "Spring Cloud Gateway diagram")

官方样例：<https://spring.io/guides/gs/gateway>

```bash
git clone https://github.com/spring-guides/gs-gateway.git
cd gs-gateway/initial
# 使用idea打开
```

编写简单的gateway处理，路由到httpbin.org（一个公共的实验站点）。

```java
@Bean
public RouteLocator myRoutes(RouteLocatorBuilder builder) {
    return builder.routes()
        .route(p -> p
            .path("/get")
            .filters(f -> f.addRequestHeader("Hello", "World"))
            .uri("http://httpbin.org:80"))
        .build();
}
```

示例解释：代理httpbin.org/get请求，新增了一个Hello\:World消息头字段。


> 这种gateway跟nginx的区别？nginx不能做吗？

> gateway相对更灵活，可编程，内置有权限、服务熔断等功能，可以做的事情跟多些。&#x20;


**rpc服务**

rpc服务采用阿里的sofa-rpc，示例下载：<https://help.aliyun.com/document_detail/149866.html?spm=a2c4g.152618.0.0.754c480bbGsKic>

rpc调用涉及client和server，server提供interface实现，client调用interface。这里隐含了client和server的工程内需分别放一个interface文件。

工程目录如下：

```bash
.
├── myclient-app  # 客户端
│   ├── app
│   │   ├── endpoint
│   │   │   ├── pom.xml
│   │   │   └── src
│   │   │       └── main
│   │   │           ├── java
│   │   │           │   └── com
│   │   │           │       └── alipay
│   │   │           │           └── samples
│   │   │           │               └── rpc
│   │   │           │                   ├── SampleRestFacade.java
│   │   │           │                   └── SampleService.java  # 接口文件
│   │   │           └── resources
│   │   └── web
│   │       ├── pom.xml
│   │       └── src
│   │           ├── main
│   │           │   ├── java
│   │           │   │   └── com
│   │           │   │       └── alipay
│   │           │   │           └── mytestsofa
│   │           │   │               ├── ReferenceHolder.java  # 获取SampleService实例引用
│   │           │   │               └── SOFABootWebSpringApplication.java  # 拉起服务，调用SampleService接口
│   │           │   └── resources
│   │           │       ├── config
│   │           │       │   ├── application-dev.properties
│   │           │       │   ├── application.properties
│   │           │       │   └── application-test.properties
│   │           │       ├── logback-spring.xml
│   │           │       └── static
│   │           │           └── index.html
│   │           └── test
│   │               └── java
│   └── pom.xml
└── myserver-app  # 服务端
    ├── app
    │   ├── endpoint
    │   │   ├── pom.xml
    │   │   └── src
    │   │       └── main
    │   │           ├── java
    │   │           │   └── com
    │   │           │       └── alipay
    │   │           │           └── samples
    │   │           │               └── rpc
    │   │           │                   ├── impl
    │   │           │                   │   ├── SampleRestFacadeImpl.java
    │   │           │                   │   └── SampleServiceImpl.java  # SampleService实现
    │   │           │                   ├── SampleRestFacade.java
    │   │           │                   └── SampleService.java  # 如你所见，这里也有一个SampleService接口文件
    │   │           └── resources
    │   │               └── META-INF
    │   │                   └── sofa-rpc
    │   │                       └── sofa-config.json
    │   └── web
    │       ├── pom.xml
    │       └── src
    │           ├── main
    │           │   ├── java
    │           │   │   └── com
    │           │   │       └── alipay
    │           │   │           └── mytestsofa
    │           │   │               └── SOFABootWebSpringApplication.java  # 仅做拉起服务的动作，为的是让SampleService实例暴露出去
    │           │   └── resources
    │           │       ├── config
    │           │       │   ├── application-dev.properties
    │           │       │   ├── application.properties
    │           │       │   └── application-test.properties
    │           │       ├── logback-spring.xml
    │           │       └── static
    │           │           └── index.html
    │           └── test
    │               └── java
    │                   └── com
    │                       └── alipay
    │                           └── mytestsofa
    │                               └── web
    └── pom.xml

```

本地编译需先配置\~/.m2/settings.xml指向阿里maven仓库（central仓库是没有pom表描述的这些包的）：<https://help.aliyun.com/document_detail/133192.html?spm=a2c4g.149866.0.0.3a17480bTRbEmC>

> 新版maven默认要求https，这里的settings源为http，可能会碰到网络问题，请修改`external:https:*`中的https为dummy


编译好会得到两个可执行jar包，启动方式：

```bash
java -jar myserver-xxxx-executable.jar
java -jar myclient-xxxx-executable.jar
```

**rpc client集成gateway**

思路，定义一个filter，从SampleService获取相关字段，然后concat到httpbin响应里。

这里碰到问题，sofaboot应用有自己的依赖，和spring cloud gateway依赖似乎有冲突，尝试集成二者存在包不匹配导致类找不到的情况。

这块作为遗留问题，以后有时间再解。

**使用spring cloud**

尝试做一次spring cloud组件集成，包括：

*   配置中心：spring cloud config
*   注册中心：spring cloud eureka
*   rest接口：spring cloud web
*   数据映射：mybatis
*   数据库：mysql

1、配置中心效果：

product和order服务[从github上](https://github.com/StaveWu/config-repository)读取配置，并本地运行。

config-repository/productService-dev.properties：

```properties
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/experiment?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8&useSSL=false
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.type=com.zaxxer.hikari.HikariDataSource
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.maximum-pool-size=15
spring.datasource.hikari.auto-commit=true
spring.datasource.hikari.idle-timeout=30000
spring.datasource.hikari.pool-name=DatebookHikariCP
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.connection-test-query=SELECT 1
mybatis.mapper-locations=classpath:mapper/*.xml
```

本地product配置：

```properties
spring.application.name=productService
eureka.client.service-url.defaultZone=http://localhost:8764/eureka/
spring.cloud.config.discovery.enabled=true
spring.cloud.config.discovery.service-id=CONFIGCENTER
spring.cloud.config.profile=dev
spring.config.import=configserver:http://localhost:8096
```

上述8096端口有配置中心服务在监听：

```properties
server.port=8096
spring.application.name=configCenter
eureka.client.service-url.defaultZone=http://localhost:8764/eureka/
spring.cloud.config.server.git.uri=https://github.com/StaveWu/config-repository.git
spring.cloud.config.server.git.username=stavewu
spring.cloud.config.server.git.password=******
```

配置中心服务主要用到了spring cloud config组件Config Server，注解：`@EnableConfigServer`

product使用的是Config Client，无需注解。

这样就形成了一条从`product -> config server:8096 -> github/config-repository`的通路。

config-repository里有好几个配置文件，product服务怎么知道要拿哪一个？

这里猜测是一种约定：基于spring.appliction.name + spring.cloud.config.profile命名的配置文件将被读取，product服务对应到productService-dev.properties。

2、mybatis效果：

在product配置文件里，通过`mybatis.mapper-locations=classpath:mapper/*.xml`读取到位于resources/mapper下的配置文件；通过`spring.datasource.url=jdbc:mysql://127.0.0.1:3306/experiment?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8&useSSL=false`连接到名为experiment的数据库实例，与配置文件配合形成数据库数据读取加载。
