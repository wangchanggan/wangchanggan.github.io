---
layout:     post
title:      Kubernetes CSI
date:       2023-01-31
catalog: true
tags:
    - Kubernetes
---

# 概述
与容器对接的存储接口标准，存储提供方只需基于标准接口进行存储插件的实现，就能使用Kubernetes的原生存储机制为容器提供存储服务了。这套标准被称为CSI（容器存储接口）。

在CSI成为Kubernetes的存储供应标准之后，存储提供方的代码就能与Kubernetes代码彻底解耦，部署也与Kubernetes核心组件分离。
# 组件

![](/img/in-post/Kubernetes/CSI-components.png)

## Controller
主要功能是提供存储服务视角对存储资源和存储卷进行管理和操作。
* external-provisioner：一个sidecar容器，监视用户创建的PersistentVolumeClaim对象，通过调用CSI驱动程序的CreateVolume和DeleteVolume函数来动态调配卷。
* external-attacher：一个sidecar容器，监视由AD控制管理器创建的VolumeAttachment对象，通过调用CSI驱动程序的ControllerPublish和ControllerUnpublish函数将卷附加到节点或从节点分离卷。
* external-resizer：一个sidecar容器，监视PersistentVolumeClaim更新，并在用户请求对PersistentVolumeClaim对象进行更多存储时触发ControllerExpandVolume对CSI端点的操作。
* CSI Driver存储驱动容器：由第三方存储提供商提供，需要实现相关接口。

上述sidecar容器通过Socket调用CSI Driver容器的CSI接口，并使用gPRC协议进行通信，CSI Driver容器负责具体的存储卷操作。
## Node

* node-driver-registrar：一个sidecar容器，它使用Kubelet插件注册机制向Kubelet注册CSI驱动程序。Kubelet负责发出CSI调用。节点驱动程序注册器向Kubelet注册CSI驱动程序，以便它知道要在哪个Unix域套接字上发出CSI调用。
* CSI Driver存储驱动容器：由第三方存储提供商提供，主要功能是接收kubelet的调用，需要实现一系列与Node相关的CSI接口。

# PVC
## 创建

![](/img/in-post/Kubernetes/pvc-create-process.png)

1. 用户创建了一个包含 PVC 的 Pod，该 PVC 要求使用动态存储卷；
2. Scheduler 根据 Pod 配置、节点状态、PV 配置等信息，把 Pod 调度到一个合适的 Worker 节点上；
3. external-provisioner监测到PVC的创建，对CSI卷驱动程序发出CreateVolume调用以创建存储卷（容量将取自PersistentVolumeClaim对象，参数将从StorageClass传递）,并创建PV，且绑定PVC。
4. AD 控制器监测到PV的创建，则创建VolumeAttachment资源。
5. external-attacher监视到VolumeAttachment的创建，触发CSI的ControllerPublishVolume 操作挂接存储卷到特定节点。此时Volume处于 NODE_READY 状态，即Node可以感知到 Volume，但是容器内依然不可见。
6. 在节点上，Kubelet中的Volume Manager等待存储设备挂接完成，执行MountDevice操作，调用Node Service的NodeStageVolume方法。主要实现对Volume格式化，并挂载到全局目录：/var/lib/kubelet/pods/[pod_uid]/volumes/kubernetes.io~iscsi/[PVname]（以 iscsi 为例）；
7. 接着执行 SetUp 操作，调用 Node Service 的 NodePublishVolume 方法将已挂载到宿主机全局目录的卷bind-mount到容器中。

## 扩容
1. external-resizer watch pvc对象，当发现pvc.Spec.Resources.Requests.storgage比pvc.Status.Capacity.storgage大，于是调csi plugin的ControllerExpandVolume方法进行controller端扩容，进行底层存储扩容，并更新pv.Spec.Capacity.storgage。
2. kubelet发现pv.Spec.Capacity.storage大于pvc.Status.Capacity.storage，于是调csi node端扩容，对dnode上文件系统扩容，成功后kubelet更新pvc.Status.Capacity.storage。