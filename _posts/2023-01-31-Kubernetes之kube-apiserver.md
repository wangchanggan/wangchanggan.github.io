---
layout:     post
title:      Kubernetes kube-apiserver
date:       2023-01-31
catalog: true
tags:
    - Kubernetes
---

# 架构设计
kube-apiserver组件负责将kubernetes的“资源组、资源版本、资源”以RESTful风格的形式对外暴露并提供服务，是Kubernetes系统集群中所有组件沟通的桥梁。

kube-apiserver提供了3种HTTP Server 服务，分别是APIExtensionsServer、KubeAPIServer、 AggregatorServer。

![](/img/in-post/Kubernetes/kube-apiserver-http-server.png)

* APIExtensionsServer：API 扩展服务(扩展器)。该服务提供了CRD(Custom Resource Definitions)自定义资源服务，开发者可通过CRD对Kubernetes资源进行扩展。该服务通过CustomResourceDefinitions对象进行管理，并通过extensionsapiserver.Scheme资源注册表管理CRD相关资源。
* AggregatorServer：API聚合服务(聚合器)。该服务提供了AA(APIAggregator)聚合服务，开发者可通过AA对Kubernetes聚合服务进行扩展，例如，metrics-server 是Kubernetes系统集群的核心监控数据的聚合器，它是AggregatorServer 服务的扩展实现。API聚合服务通过APIAggregator对象进行管理，并通过aggregatorscheme.Scheme资源注册表管理AA相关资源。
* KubeAPIServer：API核心服务。该服务提供了Kubernetes 内置核心资源服务，不允许开发者随意更改相关资源。API 核心服务通过Master对象进行管理，并通过legacyscheme.Scheme资源注册表管理Master相关资源。

APlExtensionsServer扩展服务和AggregatorServer 聚合服务都是可以在不修改Kubermetes核心代码的前提下扩展Kubernetes API的方式。只有KubeAPIServer核心服务是Kubernetes系统运行的基础，不建议随意修改它。

无论是APIExtensionsServer、KubeAPIServer还是AggregatorServer，它们都在底层依赖于GenericAPIServer。 通过GenericAPIServer 可以将Kubernetes资源与REST API进行映射。
# 启动流程
## 资源注册
将Kubernetes 所支持的资源注册到Scheme资源注册表中，这样后面启动的逻辑才能够从Scheme资源注册表中拿到资源信息并启动和运行APIExtensionsServer、KubeAPIServer、AggregatorServer这3种服务。
## Authentication认证配置
kube-apiserver作为Kubernetes集群的请求入口，接收组件与客户端的访问请求，每个请求都需要经过认证(Authentication)、 授权(Authorization)及准入控制器(Admission Controller)3个阶段，之后才能真正地操作资源。
kube-apiserver目前提供了9种认证机制，分别是BasicAuth、ClientCA、TokenAuth、BootstrapToken、RequestHeader、Webhook TokenAuth、Anonymous、OIDC、ServiceAccountAuth。每一种认证机制被实例化后会成为认证器(Authenticator),每一个认证器都被封装在http.Handler请求处理函数中，它们接收组件或客户端的请求并认证请求。
## Authorization授权配置
kube-apiserver目前提供了6种授权机制，分别是AlwaysAllow、AlwaysDeny、Webhook、Node、 ABAC、RBAC。
* AlwaysAllow：允许所有请求。
* AlwaysDeny：阻止所有请求。
* ABAC：即Attribute-Based Access Control，基于属性的访问控制。
* Webhook：基于Webhook的一种HTTP 协议回调，可进行远程授权管理。
* RBAC：即Role-Based Access Control，基于角色的访问控制。
* Node: 节点授权，专门授权给 kubelet发出的 API请求。

### RBAC
kube-apserver通过角色（Role）、集群角色（ClusterRole)、角色绑定（RoleBinding）及集群角色绑定(ClusterRoleBinding)来描述RBAC关系。

![](/img/in-post/Kubernetes/rbac-relationship.png)

* 角色(Role)：角色是一组用户的集合，与规则相关联。角色只能被授予某一个命名空间的权限。
* 集群角色(ClusterRole)：集群角色是组用户的集合，与规则相关联。集群角色能够被授子集群范围的权限，例如节点、非资源类型的服务端点(Endpoint)、跨所有命名空间的权限等。
* 规则：(PolicyRule) 规则相当于操作权限，权限控制资源的操作方法(即Verbs)。
* 角色绑定(RoleBinding)：将角色中定义的权限授予一个或者一组用户，角色只能被授予某一个命名空间的权限。
* 集群角色绑定(ClusterRoleBinding): 将集群角色中定义的权限授予一个或者一组用户，角色能够被授予集群范围的权限。

## Admission准入控制器配置
Kubernetes系统组件或客户端请求通过授权阶段之后，会来到准入控制器阶段，它会在认证和授权请求之后，对象被持久化之前，拦截kube-apiserver的请求，拦截后的请求进入准入控制器中处理，对请求的资源对象进行自定义(校验、修改或拒绝)等操作。

kube-apiserver支持多种准入控制器机制，并支持同时开启多个准入控制器功能，如果开启了多个准入控制器，则按照顺序执行准入控制器。

kube-apiserver目前提供了31种准入控制器。

# 请求流程

![](/img/in-post/Kubernetes/kube-apiserver-request-process.png)

1. 使用kubectl工具向Kubernetes API Server发起创建 Pod资源对象的请求。
2. Kubernetes API Server验证请求并将其持久保存到Etcd 集群中。 
3. Kubernetes API Server基于Watch 机制通知 kube-scheduler 调度器。 
4. kube-scheduler调度器根据预选和优选调度算法为 Pod资源对象选择最优的节点并通知 Kubernetes API Server。 
5. Kubernetes API Server将最优节点持久保存到 Etcd集群中。 
6. Kubernetes API Server通知最优节点上的 kubelet组件。 
7. kubelet 组件在所在的节点上通过与容器进程交互创建容器。 
8. kubelet 组件将容器状态上报至Kubernetes API Server. 
9. Kubernetes API Server将容器状态持久保存到Etcd集群中。