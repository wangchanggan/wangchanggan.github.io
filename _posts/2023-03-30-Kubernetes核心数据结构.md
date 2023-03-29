---
layout:     post
title:      Kubernetes核心数据结构
date:       2023-03-30
catalog: true
tags:
    - Kubernetes
---

# Group、Version、Resource 核心数据结构
在整个Kubernctes体系架构中，资源是 Kubernetcs最重要的概念。Kubernetes系统虽然有相当复杂和众多功能，但它本质上是一个资源控制系统——注册、管理、调度资源并维护资源的状态。
Kubcrnctes将资源再次分组和版本化，形成 Group（资源组）、Version（资源版本）、Resource（资源）。
* Group：资源组，在Kubernetes API Server 中也可称其为 APIGroup
* Version：资源版本，在 Kubernetes APl Server中也可称其为APIVersions
* Resource：资源，在Kubenetes APl Server 中也可称其为 APlResource
* Kind:资源种类，描述Resource的种类，与 Resource为同一级别。

![](/img/in-post/Kubernetes/data-structure.png)

Kubernetes系统支持多个Group，每个 Group支持多个 Version，每个Version支持多个 Resource，其中部分资源同时会拥有自己的子资源(即SubResource)。例如，Deployment资源拥有Status子资源。
资源组、资源版本、资源、子资源的完整表现形式为<group>/<version>/<resource>/<subresource>。以常用的Deployment资源为例，其完整表现形式为apps/v1/deployments/status。

另外，资源对象（Resource Object）由“资源组+资源版本+资源种类”组成，并在实例化后表达一个资源对象，例如 Deployment资源实例化后拥有资源组、资源版本及资源种类，其表现形式为<group>/<version>, Kind=<kind>，例如apps/v1, Kind=Deployment。

每一个资源都拥有一定数量的资源操作方法（即 Verbs），资源操作方法用于Etcd集群存储中对资源对象的增、删、改、查操作。目前Kubernetes系统支持8种资操作方法，分别是 create、delete、deletecollection、get、list、patch、update、watch操作方法。

每一个资源都至少有两个版本，分别是外部版本(External Version)和内部版本(Internal Version)。外部版本用于对外暴露给用户请求的接口所使用的资源对象。内部版本不对外暴露，仅在Kubernetes API Server内部使用。

注：资源版本与资源外部版本/内部版本属于不同的概念。

Kubernetes 资源也可分为Kubernetes Resource (Kubernetes 内置资源)和Custom Resource (自定义资源)。开发者通过CRD (即Custom Resource Definitions) 可实现自定义资源，它允许用户将自己定义的资源添加到Kubernetes系统中，并像使用Kubernetes内置资源一样使用它们。

## Group
Group（资源组)，在Kubernetes API Server中也可称其为 APIGroup. Kubernetes系统中定义了许多资源组，这些资源组按照不同功能将资源进行了划分，资源组特点如下。
* 将众多资源按照功能划分成不同的资源组，并允许单独启用/禁用资源组。当然也可以单独启用/禁用资源组中的资源。
* 支持不同资源组中拥有不同的资源版本。这方便组内的资源根据版本进行迭代升级。
* 支持同名的资源种类（即 Kind)存在于不同的资源组内。
* 资源组与资源版本通过Kubernetes API Server对外暴露，允许开发者通过HTTP 协议进行交互并通过动态客户端(即 DynamicClient)进行资源发现。
* 支持 CRD自定义资源扩展。
* 用户交互简单，例如在使用 kubectl命令行工具时，可以不填写资源组名称。

| APIGroup                                                                                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| TypeMeta<br/>Name string<br/>Versions []GroupVersionForDiscovery<br/>PreferredVersion GroupVersionForDiscovery<br/>ServerAddressByClientCIDRs []ServerAddressByClientCIDR |

* Name：资源组名称。
* Versions：资源组下所支持的资源版本。
* Preferred Version:首选版本。当一个资源组内存在多个资源版本时，Kubernetes API Server在使用资源时会选择一个首选版本作为当前版本。

在当前的Kubernetes系统中，支持两类资源组，分别是拥有组名的资源组和没有组名的资源组。
* 拥有组名的资源组：其表现形式为<group>/<version>/<resource>，例如apps/v1/deployments。
* 没有组名的资源组：被称为Core Groups（即核心资源组）或Legacy Groups，也可被称为GroupLess（即无组）。其表现形式为/<version>/<resource>，例如/v1/pods。

两类资源组表现形式不同，形成的HTTP PATH 路径也不同。拥有组名的资源组的以/apis为前缀，其表现形式为/apis/<group>/<version>/<resource>，例如http://localhost:8080/apis/apps/v1/deployments。没有组名的资源组的以/api为前缀，其表现形式为/api/<version>/<resource>，例如http://localhost:8080/api/v1/pods。

## Version

| APIVersions |
|-------------------------------------------------------------------------------------------|
| TypeMeta<br/>Versions []string<br/>ServerAddressByClientCIDRs []ServerAddressByClientCIDR |

Kubernetes的资源版本控制类似于语义版本控制(Semantic Versioning)，在该基础上的资源版本定义允许版本号以v开头，例如v1betal。每当发布新的资源时，都需要对其设置版本号，这是为了在兼容旧版本的同时不断升级新版本，这有助于帮助用户了解应用程序处于什么阶段，以及实现当前程序的迭代。语义版本控制应用得非常广泛，目前也是开源界常用的一种版本控制规范。

Kubernetes 的资源版本控制可分为3种，分别是Alpha、Beta、 Stable，它们之间的迭代顺序为Alpha→Beta→Stable，其通常用来表示软件测试过程中的3个阶段。
### Alpha版本
Alpha版本为内部测试版本，用于Kubernetes开发者内部测试，该版本是不稳定的，可能存在很多缺陷和漏洞，官方随时可能会放弃支持该版本。在默认的情况下，处于Alpha版本的功能会被禁用。Alpha版本名称一般为v1alpha1、vlalpha2、v2alpha1等。
### Beta版本
Beta版本为相对稳定的版本，Beta 版本经过官方和社区很多次测试，当功能迭代时，该版本会有较小的改变，但不会被删除。在默认的情况下，处于Beta版本的功能是开启状态的。Beta 版本命名一般为v1beta1、vlbeta2、v2beta1。
### Stable版本
Stable版本为正式发布的版本，Stable版本基本形成了产品，该版本不会被删除。在默认的情况下，处于Stable版本的功能全部处于开启状态。Stable 版本命名一般为v1、v2、v3。

## Resource
一个资源被实例化后会表达为一个资源对象(即Resource Obiect)。在 Kubernetes系统中定义并运行着各式各样的资源对象，所有资源对象都是 Entity。Kubernetes使用这些Entity来表示当前状态。可以通过 Kubernetes API Server进行查询和更新每一个资源对象。
Kubernetes 目前支持两种 Entity，分别介绍如下。
* 持久性实体(Persistent Entity)：在资源对象被创建后，Kubernetes会持确保该资源对象存在。大部分资源对象属于持久性实体，例如 Deployment资源对象。
* 短暂性实体(Ephemeral Entity)：也可称其为非持久性实体(Non-PersistenEntity)。在资源对象被创建后，如果出现故障或调度失败，不会重新创建该资源对象，例如 Pod资源对象。


| APIResource                                                                                                                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Name string<br/>SingularName string<br/>Namespaced bool<br/>Group string<br/>Version string<br/>Kind string<br/>Verbs Verbs<br/>ShortNames []string<br/>Categories []string<br/>StorageVersionHash string |

* Name：资源名称。
* SingularName:资源的单数名称，它必须由小写字母组成，默认使用资源种类(Kind)的小写形式进行命名。例如，Pod 资源的单数名称为pod，复数名称为pods。
* Namespaced：资源是否拥有所属命名空间。
* Group：资源所在的资源组名称。
* Version：资源所在的资源版本。
* Kind：资源种类。
* Verbs：资源可操作的方法列表，例如get、list、 delete、 create、 update等。
* ShortNames：资源的简称，例如Pod资源的简称为po。

### 资源外部版本与内部版本

* External Object：外部版本资源对象，也称为Versioned Object（即拥有资源版本的资源对象)。外部版本用于对外暴露给用户请求的接口所使用的资源对象，例如，用户在通过YAML 或JSON格式的描述文件创建资源对象时，所使用的是外部版本的资源对象。外部版本的资源对象通过资源版本（Alpha、 Beta、Stable）进行标识。
* Internal Object：内部版本资源对象。内部版本不对外暴露，仅在Kubernetes API Server 内部使用。内部版本用于多资源版本的转换，例如将v1beta版本转换为v1版本，其过程为 v1beta1→internal→v1，即先将v1beta1转换为内部版本(internal)，再由内部版本(internal)转换为v1版本，内部版本资源对象通过runtime.APIVersionInternal (即_ internal) 进行标识。

注意：资源版本(如v1beta1、v1等)与外部版本/内部版本概念不同。拥有资源版本的资源属于外部版本，拥有runtime.APIVerionlnternal标识的资源属于内部版本。

![](/img/in-post/Kubernetes/resource-external-and-internal-version.png)

资源的外部版本和内部版本是需要相互转换的，而用于转换的函数需要事先初始化到资源注册表(Scheme) 中。多个外部版本(External Version)之间的资源进行相互转换，都需要通过内部版本(Internal Version)进行中转。这也是Kubemetes能实现多资源版本转换的关键。

外部版本的资源需要对外暴露给用户请求的接口，所以资源代码定义了JSON Tags和Proto Tags，用于请求的序列化和反序列化操作。内部版本的资源不对外暴露，所以没有任何的JSON Tags和ProtoTags定义。
### 资源操作方法

![](/img/in-post/Kubernetes/resource-operation-method.png)

### 资源与命名空间
Kubernetes系统支持命名空间(Namespaces)，其用来解决Kubernetes集群中资源对象过多导致管理复杂的间题。每个命名空间相当手一个“虚拟集群”，不同命名空间之间可以进行隔离，当然也可以通过某种方式跨命名空间通信。
Kubernetes 系统中默认内置了4个命名空间：
* default：所有未指定命名空间的资源对象都会被分配给该命名空间。
* kube-system：所有由Kubernetes系统创建的资源对象都会被分配给该命名空间。
* kube-public：此命名空间下的资源对象可以被所有人访问(包括未认证用户)。
* kube-node-lease：此命名空间下存放来自节点的心跳记录(节点租约信息)。

### 自定义资源
Kubernetes系统拥有强大的高扩展功能，其中自定义资源(Custom Resource)就是一种常见的扩展方式，即可将自己定义的资源添加到Kubernetes系统中。Kubernetes系统附带了许多内置资源，但是仍有些需求需要使用自定义资源来扩展Kubernetes的功能。

在Kubernetes系统早期，是通过ThirdPartyResource (TPR) 来实现扩展自定义资源的，但自Kubermetes 1.7版本后，ThirdPartyResource功能已被弃用，并且其已根据Beta功能的弃用政策在Kubernetes 1.8中被删除，取而代之的是CustomResourceDefinitions (自定义资源定义，简称CRD)。

开发者通过CRD可以实现自定义资源，它允许用户将自己定义的资源添加到Kubernetes 系统中，并像使用Kubernetes 内置资源一样使用这些资源， 例如，在YAML/JSON文件中带有Spec的资源定义都是对Kubernetes中的资源对象的定义，所有的自定义资源都可以与Kubernetes系统中的内置资源一样使用 kubectl或client-go进行操作。

### 资源对象描述文件定义
Kubernetes资源可分为内置资源(Kubernetes Resources)和自定义资源(Custom Resources)，它们都通过资源对象描述文件(Manifest File)进行定义。

![](/img/in-post/Kubernetes/resource-object-manifest-file.png)

一个资源对象需要用 5个字段来描述它，分别是Group/Version、Kind、MetaData、Spec、Status，这些字段定义在YAML或JSON文件中。Kubernetes 系统中的所有的资源对象都可以采用YAML或JSON格式的描述文件来定义。
* apiVersion：指定创建资源对象的资源组和资源版本，其表现形式为<group>/<version>，若是core资源组(即核心资源组)下的资源对象，其表现形式为<version>。
* Kind：指定创建资源对象的种类。
* metadata：描述创建资源对象的元数据信息，例如名称、命名空间等。
* spec：包含有关Deployment资源对象的核心信息，告诉Kubernetes期望的资源状态、副本数量、环境变量、卷等信息。
* status：包含有关正在运行的Deployment资源对象的信息。

每一个Kubernetes 资源对象都包含两个嵌套字段，即spec字段和status字段。其中spec字段是必需的，它描述了资源对象的“期望状态”(Desired State)，而status字段用于描述资源对象的“实际状态”(Actual State), 它是由Kubernetes 系统提供和更新的。在任何时刻，Kubernnetes 控制器一直努力地管理着对象的实际状态以与期望状态相匹配。

# runtime.Object类型基石
Runtime被称为“运行时”，它一般指程序或语言核心库的实现。Kubernetes Runtime提供了通用资源类型runtime.Object。

runtime.Object是 Kubernetes类型系统的基石。Kubenetes上的所有资源对象(Resource Object)实际上就是一种Go语言的Struct类型，相当于一种数据结构，它们都有一个共同的结构叫runtime.Object。runtime.Object被设计为Interface接口类型，作为资源对象的通用资源对象。

# Unstructured数据
无法预知数据结构的数据类型或属性名称不确定的数据类型是非结构化数据，其无法通过构建预定的struct数据结构来序列化或反序列化数据。

Kubernetes非结构化数据通过map[string]interface{}表达，并提供接口。在client-go编程式交互的DynamicClient内部，实现了Unstructured类型，用于处理非结构化数据。

# Scheme资源注册表
Kubernetes系统拥有众多资源，每一种资源就是一个资源类型，这些资源类型需要有统一的注册、存储、查询、管理等机制。目前Kubernetes系统中的所有资源类型都已注册到Scheme资源注册表中，其是一个内存型的资源注册表，拥有如下特点。
* 支持注册多种资源类型，包括内部版本和外部版本。
* 支持多种版本转换机制。
* 支持不同资源的序列化/反序列化机制。

Scheme资源注册表支持两种资源类型(Type)的注册，分别是UnversionedType和 KnownType资源类型，分别介绍如下。
* UnversionedType：无版本资源类型，这是一个早期Kubemetes系统中的概念，它主要应用于某些没有版本的资源类型，该类型的资源对象并不需要进行转换。在目前的Kubernetes发行版本中，无版本类型已被弱化，几乎所有的资源对象都拥有版本，但在 metav1元数据中还有部分类型，它们既属于meta.k8s.io/v1又属于UnversionedType无版本资源类型，例如metav1.Status.metav1.APIVersions、metav1.APIGroupList、metav1.APIGroup、metav1.APIResourceList。
* KnownType：是目前Kubernetes最常用的资源类型，也可称其为“拥有版本的资源类型”。

在 Scheme 资源注册表中，UnversionedType资源类型的对象通过scheme.AddUnversionedTypes方法进行注册，KnownType资源类型的对象通过scheme.AddKnownTypes方法进行注册。

# Codec编解码器
* Serializer：序列化器，包含序列化操作与反序列化操作。序列化操作是将数据（例如数组、对象或结构体等)转换为字符串的过程，反序列化操作是将字符串转换为数据的过程，因此可以轻松地维护数据结构并存储或传输数据。
* Codec:编解码器，包含编码器与解码器。编解码器是一个通用术语，指的是可以表示数据的任何格式，或者将数据转换为特定格式的过程。所以，可以将 Serializer序列化器也理解为Codec 编解码器的一种。

![](/img/in-post/Kubernetes/codec.png)

Codec编解码器包含3种序列化器，在进行编解码操作时，每一种序列化器都对资源对象的 metav1.TypeMeta(即 APIVersion和Kind字段)进行验证，如果资源对象未提供这些字段，就会返回错误。每种序列化器分别实现了Encode序列化方法与Decode反序列化方法：
* jsonSerializer：JSON格式序列化/反序列化器。它使用 application/json的ContentType作为标识。
* yamlSerializer：YAML格式序列化/反序列化器。它使用 application/yaml的ContentType作为标识。
* protobufSerializer：Protobuf 格式序列化/反序列化器。它使用application/vnd.kubernetes.protobuf的ContentType作为标识。

# Converter资源版本转换器
在Kubernetes系统中，同一资源拥有多个资源版本，Kubernetes系统允许同一资源的不同资源版本进行转换，例如Deployment资源对象，当前运行的是v1beta1资源版本，但v1beta1 资源版本的某些功能或字段不如v1资源版本完善，则可以将Deployment资源对象的v1beta1资源版本转换为v1版本。可通过kubectl convert命令进行资源版本转换。

Converter资源版本转换器主要用于解决多资源版本转换问题，Kubernetes 系统中的一个资源支持多个资源版本，通过内都版本(Internal Version) 机制实现资源版本转换。

![](/img/in-post/Kubernetes/resource-version-conversion.png)

当需要在两个资源版本之间转换时，Converter资源版本转换器先将第一个资源版本转换为_internal内部版本，再转换为相应的资源版本。每个资源只要能支持内部版本，就能与其他任何资源版本进行间接的资源版本转换。