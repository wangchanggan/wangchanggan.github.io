---
layout:     post
title:      Kubernetes client-go
date:       2023-01-31
catalog: true
tags:
    - Kubernetes
---

# Client客户端对象
client-go支持4种Client客户端对象与Kubernetes API Server交互的方式。
![](/img/in-post/Kubernetes/client-go-objects.png)

* RESTClient是最基础的客户端。RESTClient对 HTTP Request进行了封装，实现了 RESTful 风格的 API。ClientSet、DynamicClient及DiscoveryClient客户端都是基于RESTClient 实现的。
* ClientSet在RESTClient的基础上封装了对Resource和Version的管理方法。每一个 Resource可以理解为一个客户端，而ClientSet则是多个客户端的集合，每一个Resource和Version都以函数的方式暴露给开发者。ClientSet只能够处理Kubernetes内置资源。
* DynamicClient与ClientSet最大的不同之处是，ClientSet仅能访问Kubernetes自带的资源(即 Client集合内的资源)，不能直接访问CRD自定义资源。DynamicClient能够处理Kubernetes中的所有资源对象，包括Kubernetes内置资源与CRD自定义资源。

ClientSet需要预先实现每种Resource和 Version的操作，其内部的数据都是结构化数据(即已知数据结构)。而 DynamicClient内部交给Unstructured来承载，用于处理非结构化数据结构(即无法提前预知数据结构)，使用嵌套的 map[string]-interface{] 结构存储 Kubernetes APIServer 的返回值，利用反射机制在运行时进行数据绑定，松耦合意味着更高的灵活性，但无法获取强数据类型检查和验证的好处。
* DiscoveryClient发现客户端，用于发现 kube-apiserver所支持的资源组、资源版本、资源信息（即 Group、Versions、Resources）。DiscoveryClient可以将资源相关信息存储于本地，默认存储位置为~/.kube/cache和~/.kube/http-cache。缓存可以减轻client-go对Kubernetes API Server的访问压力。默认每10分钟与Kubernetes API Server同步一次，同步周期较长，因为资源组、资源版本、资源信息一般很少变动。

以上4种客户端都可以通过kubeconfig 配置信息连接到指足的Kubernetes APl Server。

## kubeconfig配置管理
kubeconfig用于管理访问 kube-apiserver的配置信息，存储了集群、用户、命名空间和身份验证等信息，在默认的情况下，kubeconfig存放在$HOME/.kube/config路径下。

kubeconfig配置信息通常包含3个部分：
* clusters：定义Kubernetes集群信息，例如kube-apiserver的服务地址及集群的证书信息等。
* users：定义Kubernetes 集群用户身份验证的客户端凭据，例如client-certificate、client-key、token 及usermame/password等。
* contexts：定义Kubernetes 集群用户信息和命名空间等，用于将请求发送到指定的集群。

# Informer机制
在Kubernetes系统中，组件之间通过HTTP协议进行通信，在不依赖任何中间件的情况下，通过Informer机制保证消息的实时性、可靠性和顺序性等。

![](/img/in-post/Kubernetes/informer.png)

1. 资源Informer

每一个Kubernetes资源上都实现了Informer机制。每一个Informer上都会实现Informer和Lister方法。
```
type PodInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.PodLister
}
```

2. Shared Informer共享机制

若同一资源的Informer被实例化了多次，每个Informer使用一个Reflector，那么会运行过多相同的ListAndWatch，太多重复的序列化和反序列化操作会导致Kubernetes API Server负载过重。

Shared Informer可以使同一类资源Informer共享一个Reflector，这样可以节约很多资源。通过map数据结构实现共享的Informer机制。Shared Informer 定义了一个map数据结构，用于存放所有Informer的字段。

```
type sharedInformerFactory struct {
    ...
	informers map[reflect.Type]cache.SharedIndexInformer
}

func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
	f.lock.Lock()
	defer f.lock.Unlock()

	informerType := reflect.TypeOf(obj)
	informer, exists := f.informers[informerType]
	if exists {
		return informer
	}

	resyncPeriod, exists := f.customResync[informerType]
	if !exists {
		resyncPeriod = f.defaultResync
	}

	informer = newFunc(f.client, resyncPeriod)
	f.informers[informerType] = informer

	return informer
}
```
informers字段中存储了资源类型和对应于SharedIndexInformer 的映射关系。InformerFor函数添加了不同资源的Informer，在添加过程中如果已经存在同类型的资源Informer，则返回当前Informer，不再继续添加。

## Reflector
Reflector用于监控（Watch）指定的Kubernetes资源，当监控的资源发生变化时，触发相应的变更事件，例如Added（资源添加）事件、Updated（资源更新）事件、Deleted（资源删除）事件，并将其资源对象存放到本地缓存DeltaFIFO中。

通过NewReflector实例化Reflector对象，实例化过程中须传入 ListerWatcher 数接口对象，它拥有 List和 Watch方法，用于获取及监控资源列表。

ListAndWatch 函数实现可分为两部分：
1. 获取资源列表数据

ListAndWatch List 在程序第一次运行时获取该资源下所有的对象数据并将其存储至DeltaFIFO中。
获取资源数据是由 options的 ResourceVersion（资源版本号）参数控制的，Kubernetes 中所有的资源都拥有该字段，它标识当前资源对象的版本号。每次修改当前资源对象时，Kubernetes APl Server 都会更改ResourceVersion，使得 client-go执行 Watch操作时可以根据ResourceVersion来确定当前资源对象是否发生变化。

Kubernetes API Server对 ResourceVersion资源版本号依赖于Etcd集群中的全局Index机制来进行管理的。在Etcd集群中，有两个较关键的Index：

* createdIndex：全局唯一且递增的正整数。每次在Etcd集群中创建key其会递增。
* modifiedIndex: 与createdIndex功能类似，每次在 Etcd集群中修改key时其会递增。

createdIndex 和 modifiedIndex都是原子操作，其中 modifiedIndex机制被Kubernetes系统用于获取资源版本号(ResourceVersion)。 Kubernetes系统通过资源版本号的概念来实现乐观并发控制，也称乐观锁(Optimistic Concurrency Control)。

2. 监控资源对象

Watch (监控)操作通过HTTP协议与Kubernetes API Server建立长连接，接收Kubernetes API Server发来的资源变更事件。Watch 操作的实现机制使用HTTP协议的分块传输编码(Chunked Transfer Encoding)。当client-go 调用Kubernetes API Server时，Kubernetes API Server在Response的HTTP Header中设置Transfer-Encoding的值为chunked，表示采用分块传输编码，客户端收到该信息后，便与服务端进行连接，并等待下一个数据块(即资源的事件信息)。
## DeltaFIFO
DeltaFIFO 可以分开理解，FIFO是一个先进先出的队列，它拥有队列操作的基本方法，例如Add、Update、Delete、List、Pop、Close等，而Delta是一个资源对象存储，它可以保存资源对象的操作类型，例如Added（添加）操作类型、Updated(更新）操作类型、Deleted（删除）操作类型、Sync（同步）操作类型等，消费者在处理该资源对象时能够了解该资源对象所发生的事情。
## Indexer
Indexer是 client-go用来存储资源对象并自带索引功能的本地存储，Reflector从DeltaFIFO中将消费出来的资源对象存储至indexer。Indexer与Etcd集群中的数据完全保持一致。client-go可以很方便地从本地存储中读取相应的资源对象数据，而无须每次从远程Etcd集群中读取，以减轻kubernetes API Server 和 Etcd集群的压力。

ThreadSafeMap是实现并发安全的存储。作为存储，它拥有存储相关的增、删、改、查操作方法，例如Add、Update、Delete、List、Get、 Replace、 Resync等。

Indexer在ThreadSafeMap的基础上进行了封装，它继承了与ThreadSafeMap相关的操作方法并实现了Indexer Func等功能，例如 Index、IndexKeys、GetIndexers等方法，这些方法 ThreadSafeMap提供了索引功能。
# WorkQueue
WorkQueue称为工作队列，Kubernetes的 WorkQueue队列与普通FIFO（先进先出,First-In, First-Out）队列相比，实现略显复杂，它的主要功能在于标记和去重。
## FIFO队列
## 延迟队列
延迟队列，基于FIFO队列接口封装，在原有功能上增加了AddAfter方法，其原理是延迟一段时间后再将元素插入FIFO队列。
## 限速队列
限速队列，基于延迟队列和FIFO队列接口封装，限速队列接口(RateLimitinginterface)在原有功能上增加了AddRateLimited、Forget、NumRequeues方法。限速队列的重点不在于RateLimitinginterface接口，而在于它提供的4种限速算法接口(RateLimiter)，包括令牌桶算法、排队指数算法、计数器算法和混合模式。其原理是，限速队列利用延迟队列的特性，延迟某个元素的插入时间，达到限速目的。


