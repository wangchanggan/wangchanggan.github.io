---
layout:     post
title:      Kubernetes网络之DNS
date:       2023-02-26
catalog: true
tags:
    - Kubernetes
---

# 通过域名访问服务
## DNS服务基本框架
Kubernetes DNS服务是用来解析Kubernetes集群内的Pod和Service域名的，而且一般情况下只供集群内的容器使用。

当Kubernetes的DNS服务Cluster IP分配后，安装程序会给Kubelet配置--cluster-dns=<dns service ip>启动参数，DNS服务的IP地址将在用户容器启动时传递，并写入每个容器的/etc/resolv.conf文件。

除此之外，Kubelet的--cluster_domain=<default-local-domain>参数支持配置集群域名后缀，默认是cluster.local。
## 域名解析基本原理
目前，Kubernetes DNS加载项支持正向查找（A Record）、端口查找（SRV记录）、反向IP地址查找（PTR记录）及其他功能。

对于Service，Kubernetes DNS服务器会生成三类DNS记录，分别是A记录、SRV记录和CNAME记录。
### A记录
A记录（A Record）是用于将域或子域指向某个IP地址的DNS记录的最基本类型。

Kubernetes为“normal”和“headless”服务分配不同的A Record name。“headless”服务与“normal”服务的不同之处在于它们未分配Cluster IP且不执行负载均衡。

普通Service的A记录的映射关系是：
```
{service_name}.{service_namespace}.svc.{domain} -> Cluster IP
```
每个部分字段的含义是：
* service_name：Service名；
* namespace：Service所在namespace；
* domain：提供的域名后缀，是Kubelet通过--cluster-domain配置的，比如默认的cluster.local。

在Pod中可以通过域名{service_name}.{service_namespace}.svc.{domain}访问任何服务，也可以使用缩写{service_name}.{service namespace}直接访问。如果Pod和Service在同一个namespace中，那么甚至可以直接使用{service_name}访问。

headless Service的A记录的映射关系是：
```
{service_name}.{service_namespace}.svc.{domain} -> 后端Pod IP列表
```
如果在Pod Spec指定hostname和subdomain，那么Kubernetes DNS会额外生成Pod的A记录：
```
{hostname}.{subdomain}.{pod_namespace}.pod.cluster.local -> Pod IP
```

### SRV记录
SRV记录是通过描述某些服务协议和地址促进服务发现的。SRV记录通常定义一个符号名称和作为域名一部分的传输协议（如TCP），并定义给定服务的优先级、权重、端口和目标。

Kubernetes DNS的SRV记录是按照一个约定俗成的规定实现了对服务端口的查询：
```
{port name}.{port protocol}.{service name}.{service namespace].svc.cluster.local -> Service Port
```
SRV记录是为“normal”或“headless”服务的部分指定端口创建的。SRV记录采用my-port-name.my-port-protocol.my-svc.mynamespace.svc.cluster.local的形式。对于常规服务，它被解析的端口号和域名是my-svc.my-namespace.svc.cluster.local。在“headless”服务的情况下，此name解析为多个answer，每个answer都支持服务。每个answer都包含auto-generated-name.my-svc.my-namespace.svc.cluster.local表单的Pod端口号和域名。
### CNAME记录
CNAME记录用于将域或子域指向另一个主机名。为此，CNAME使用现有的A记录作为其值。相反，A记录会解析为指定的IP地址。此外，在Kubernetes中，CNAME记录可用于联合服务的跨集群服务发现。在整个场景中会有一个跨多个Kubernetes集群的公共服务。所有Pod都可以发现这项服务（无论这些Pod在哪个集群上）。这是一种跨集群服务发现方法。
## DNS使用
Pod的默认主机名由Pod的metadata.name值定义。用户可以通过在可选的hostname字段中指定一个新值更改默认主机名。用户还可以在subdomain字段中自定义子域名。例如，在命名空间my-namespace中，将hostname设置为custom-host，将subdomain设置为custom-subdomain的Pod将具有完全限定的域名（FQDN）custom-host.custom-subdomain.mynamespace.svc.cluster.local。
```
# cat /etc/resolv.conf
search default.pod.cluster.local default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.0.0.1
options ndots:5
```
其中，DNS Server的IP地址是10.0.0.1。options ndots:5的含义是当查询的域名字符串内的点字符数量超过ndots（5）值时，则认为是完整域名，直接解析，否则Linux系统会自动尝试用default.pod.cluster.local、default.svc.cluster.local或svc.cluster.local补齐域名后缀。
### Kubernetes域名解析策略
Kubernetes域名解析策略对应到Pod配置中的dnsPolicy，有4种可选策略：
* None：从Kubernetes 1.9版本起引入的一个新选项值。它允许Pod忽略Kubernetes环境中的DNS设置。应使用dnsConfigPod规范中的字段提供所有DNS设置；
* ClusterFirstWithHostNet：对于使用hostNetwork运行的Pod，用户应该明确设置其DNS策略为ClusterFirstWithHostNet；
* ClusterFirst：任何与配置的群集域后缀（例如cluster.local）不匹配的DNS查询（例如“www.kubernetes.io”）将转发到从宿主机上继承的上游域名服务器。集群管理员可以根据需要配置上游DNS服务器；
* Default：Pod从宿主机上继承名称解析配置。

# Kubernetes DNS架构演进之路
## Kube-dns的工作原理
Kube-dns架构历经两个较大的变化。Kubernetes 1.3之前使用etcd+kube2sky+SkyDNS的架构，Kubernetes 1.3之后使用kubedns+dnsmasq+exechealthz的架构，这两种架构都利用了SkyDNS的能力。SkyDNS支持正向查找（A记录）、服务查找（SRV记录）和反向IP地址查找（PTR记录）。
### etcd+kube2sky+SkyDNS

* etcd：存储所有DNS查询所需的数据；
* kube2sky：观察Kubernetes API Server处Service和Endpoints的变化，然后同步状态到Kube-dns自己的etcd；
* SkyDNS：监听在53端口，根据etcd中的数据对外提供DNS查询服务。

![img.png](/img/in-post/Kubernetes/etcd-kube2sky-SkyDNS.png)

SkyDNS配置etcd作为后端数据储存，当Kubernetes cluster中的DNS请求被SkyDNS接受时，SkyDNS从etcd中读取数据，然后封装数据并返回，完成DNS请求响应。Kube2Sky通过watch Service和Endpoints更新etcd中的数据。
### kubedns+dnsmasq+exechealthz

* kubedns：从Kubernetes API Server处观察Service和Endpoints的变化并调用SkyDNS的golang库，在内存中维护DNS记录。kubedns作为dnsmasq的上游在dnsmasq cache未命中时提供DNS数据；
* dnsmasq：DNS配置工具，监听53端口，为集群提供DNS查询服务。dnsmasq提供DNS缓存，降低了kubedns的查询压力，提升了DNS域名解析的整体性能；
* exechealthz：健康检查，检查Kube-dns和dnsmasq的健康，对外提供/healthz HTTP接口以查询Kube-dns的健康状况。

![img.png](/img/in-post/Kubernetes/kubedns-dnsmasq-exechealthz.png)

* 从Kubernetes API Server那边观察（watch）到的Service和Endpoints对象没有存在etcd，而是缓放在内存中，既提高了查询性能，也省去了维护etcd存储的工作量；
* 引入了dnsmasq容器，由它接受Kubernetes集群中的DNS请求，目的就是利用dnsmasq的cache模块，提高解析性能；
* 没有直接部署SkyDNS，而是调用了SkyDNS的golang库，相当于之前的SkyDNS和Kube2Sky整合到了一个进程中；
* 增加了健康检查功能。

运行过程中，dnsmasq在内存中预留一个缓冲区（默认是1GB），保存最近使用到的DNS查询记录。如果缓存中没有要查找的记录，dnsmasq会去kubeDNS中查询，同时把结果缓存起来。

综上所述，无论是哪个版本的架构，Kube-dns的本质就是一个Kubernetes API对象的监视器+SkyDNS。
## CoreDNS
CoreDNS作为CNCF中托管的一个域名发现的项目，原生集成Kubernetes，它的目标是成为云原生的DNS服务器和服务发现的参考解决方案。从Kubernetes 1.12开始，CoreDNS就成了Kubernetes的默认DNS服务器。
## 特点

1. 插件化（Plugins）。基于Caddy服务器框架，CoreDNS实现了一个插件链的架构，将大量应用端的逻辑抽象成插件的形式（例如，Kubernetes的DNS服务发现、Prometheus监控等）暴露给使用者。
2. 配置简单化。引入表达力更强的DSL，即Corefile形式的配置文件（也是基于Caddy框架开发的）。
3. 一体化的解决方案。区别于Kube-dns“三合一”的架构，CoreDNS编译出来就是一个单独的可执行文件，内置了缓存、后端存储管理和健康检查等功能，无须第三方组件辅助实现其他功能，从而使部署更方便，内存管理更安全。

### Corefile
Corefile是CoreDNS的配置文件（源于Caddy框架的配置文件Caddyfile），它定义了：
* DNS server以什么协议监听在哪个端口（可以同时定义多个server监听不同端口）；
* DNS负责哪个zone的权威（authoritative）DNS解析；
* DNS server将加载哪些插件。

```
ZONE:[PORT]{
    [PLUGIN]
}
```

* ZONE：定义DNS server负责的zone
* PORT是可选项，默认为53；
* PLUGIN：定义DNS server要加载的插件，每个插件可以有多个参数。

![](/img/in-post/Kubernetes/corefile.png)