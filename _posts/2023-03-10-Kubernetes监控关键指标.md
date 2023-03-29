---
layout:     post
title:      Kubernetes监控关键指标
date:       2023-03-10
catalog: true
tags:
    - Kubernetes
---

# 工作负载节点
所有的 Kubernetes 组件，都提供了/metrics接口用来暴露监控数据。
## Kube-Proxy
Kube-Proxy 监听的端口10249用来暴露监控指标。
### 通用的 Go 程序相关的指标

| 通用的Go程序相关的指标 | 解释 |
| ----- | ----- |
| go_gc_duration_seconds | GC时间 |
| go_goroutines | goroutine数量 |
| go_threads | 线程数量 |
| process_max_fds | 进程可以打开的最大文件句柄数量 |
| process_open_fds | 进程当前打开了多少文件句柄 |
| process_resident_memory_bytes	| 内存使用大小 |
| process_cpu_seconds_total	| 进程使用的CPU时间 |
| process_start_time_seconds | 进程启动时间戳 |

以上指标，只要是通过 Prometheus Go SDK 埋点的程序都会有，另外包括Kubelet、APIServer、Scheduler 等。
### 请求 APIServer 的指标
Kubernetes 中多个组件都要调用 APIServer 的接口，每秒调用多少次、有多少成功多少失败、耗时情况如何，这些指标也比较关键。

比如：
* rest_client_request_duration_seconds：请求 APIServer 的耗时统计
* rest_client_requests_total：请求 APIServer 的调用量统计

### 规则同步类指标
Kube-Proxy 的核心职能，就是去 APIServer 获取转发规则，修改本地的 iptables 或者 ipvs 的规则，所以这些规则同步相关的指标至关重要。

| 规则同步类指标 | 解释 |
| ----- | ----- |
| kubeproxy_sync_proxy_rules_duration_seconds | 规则同步耗时 |
| kubeproxy_sync_proxy_rules_endpoint_changes_total	| endpoint变化的总次数 |
| kubeproxy_sync_proxy_rules_iptables_restore_failures_total | iptables restore失败总次数 |
| kubeproxy_sync_proxy_rules_last_queued_timestamp_seconds | 最近一次开始同步的时间戳 |
| kubeproxy_sync_proxy_rules_last_timestamp_seconds | 最近一次完成同步的时间戳 |

## Kubelet
Kubelet监听的端口10250用来暴露metrics 指标，暴露了两类 metrics 数据，一个是 /metrics，暴露的是 Kubelet 自身的监控数据，另一个是 /metrics/cadvisor，暴露的是容器的监控数据。
### Kubelet关键指标
Kubelet会吐出Go进程相关的通用指标以及和APIServer通信相关的度量指标。Kubelet核心职能是管理Pod，操作各种CNI、CSI相关的接口，和容器引擎打交道，度量这类操作的指标就显得尤为关键。

| kubelet关键指标 | 解释 |
| ----- | ----- |
| kubelet_docker_operations_duration_seconds | 操作Docker引擎的耗时 |
| kubelet_docker_operations_errors_total | 操作Docker引擎的失败次数 |
| kubelet_docker_operations_timeout_total | 操作Docker引擎的超时次数 |
| kubelet_docker_operations_total | 操作Docker引擎的累计次数 |
| kubelet_network_plugin_operations_duration_seconds | 网络插件操作耗时 |
| kubelet_network_plugin_operations_errors_total | 网络插件操作失败的次数 |
| kubelet_network_plugin_operations_total | 网络插件操作累计次数 |

### 容器负载指标
容器负载主要是关心 CPU、内存、网络、IO，尤其是 CPU 和内存。
* CPU 指标

```
sum(
irate(container_cpu_usage_seconds_total[3m])
) by (pod,id,namespace,container,image)
/
sum(
container_spec_cpu_quota/container_spec_cpu_period
) by (pod,id,namespace,container,image)
```
计算 CPU 使用率

| 指标 | 解释 |
| ----- | ----- |
| container_cpu_usage_seconds_total | 统计容器的 CPU 在一秒内消耗使用率，应注意的是该 container 所有的 CORE。 |
| container_spec_cpu_quota | 容器的使用 CPU 时间周期总量 |
| container_spec_cpu_period | 当对容器进行 CPU 限制时，CFS（完全公平调度策略） 调度的时间窗口，又称容器 CPU 的时钟周期，默认值是100，000 微秒。 |

其中 container_spec_cpu_quota/container_spec_cpu_period，就代表该容器有多少个 CORE，通常对应 kubernetes 的 resource.cpu.limits 的值。

假设container_spec_cpu_period设置为默认值，即 100,000 微秒，并且容器 CPU 限制设置为半个核心 (0.5)。在这种情况下：
```
container_spec_cpu_period 100,000
container_spec_cpu_quota 50,000  # =container_spec_cpu_period*0.5
```
将 CPU 限制设置为两个内核后，将拥有：
```
container_spec_cpu_quota  200,000
```

```
increase(container_cpu_cfs_throttled_periods_total[1m])
/
increase(container_cpu_cfs_periods_total[1m]) * 100
```
计算 CPU 被限制的时间比例，如果这个值很高，说明容器在使用 CPU 资源的时候经常被限制，需要提高这个容器的 CPU Quota。延迟敏感型的应用，需要特别关注这个指标。
* 内存指标

```
container_memory_working_set_bytes
/
container_spec_memory_limit_bytes
and
container_spec_memory_limit_bytes != 0
```
计算内存使用率，分子是容器的内存占用，分母是内存 Limit 大小。有些容器没有指定内存 Limit，需要有个 and 语句来做限制，只有 limit_bytes 不等于0运算才有意义。
* Pod 网络流量
```
irate(container_network_transmit_bytes_total[1m]) * 8
irate(container_network_receive_bytes_total[1m]) * 8
```
transmit 是出向，receive 是入向，这两个指标都是 Counter 类型的值，单调递增，所以使用 irate 计算每秒速率。因为网络流量一般都是用 bit 作为单位，所以最后乘以 8，把 bit换算成 byte。
* Pod 硬盘 IO 读写流量
```
irate(container_fs_reads_bytes_total[1m])
irate(container_fs_writes_bytes_total[1m])
```

# 控制面组件
## APIServer
APIServer 的核心职能是 Kubernetes 集群的 API 总入口，Kube-Proxy、Kubelet、Controller-Manager、Scheduler 等都需要调用 APIServer，所以 APIServer 的监控，完全按照 RED 方法论来梳理即可，最核心的就是请求吞吐和延迟。

| 指标 | 解释 |
| ----- | ----- |
| apiserver_request_total | 请求量的指标，可以统计每秒请求数、成功率。 |
| apiserver_request_duration_seconds | 请求耗时的指标。 |
| apiserver_current_inflight_requests | APIServer当前处理的请求数，分为 mutating（非 get、list、watch 的请求）和 readOnly（get、list、watch请求）两种，请求量过大就会被限流，所以这个指标对观察容量水位很有帮助。 |

## Controller-manager
Controller-manager 负责监听对象状态，并与期望状态做对比。如果状态不一致则进行调谐，重点关注的是任务数量、队列深度等。

| 指标 | 解释 |
| ----- | ----- |
| workqueue_adds_total | 各个 controller 接收到的任务总数。 |
| workqueue_depth | 各个 controller 的队列深度，表示各个 controller 中的任务的数量，数量越大表示越繁忙。 |
| workqueue_queue_duration_seconds | 任务在队列中的等待耗时，按照控制器分别统计。 |
| workqueue_work_duration_seconds | 任务出队到被处理完成的时间，按照控制器分别统计。 |
| workqueue_retries_total | 任务进入队列的重试次数。 |

## Scheduler
Scheduler 在 Kubernetes 架构中负责把对象调度到合适的 Node 上，在这个过程中会有一系列的规则计算和筛选，重点关注调度这个动作的相关指标。

| 指标 | 解释 |
| ----- | ----- |
| leader_election_master_status | 调度器的选主状态，1 表示 master，0 表示 backup。 |
| scheduler_queue_incoming_pods_total | 进入调度队列的 Pod 数量。 |
| scheduler_pending_pods | Pending 的 Pod 数量。 |
| scheduler_pod_scheduling_attempts | Pod 调度成功前，调度重试的次数分布。 |
| scheduler_framework_extension_point_duration_seconds | 调度框架的扩展点延迟分布，按 extension_point 统计。 |
| scheduler_schedule_attempts_total | 按照调度结果统计的尝试次数，“unschedulable”表示无法调度，“error”表示调度器内部错误。 |

## etcd
etcd 在 Kubernetes 的架构中作用巨大，相对也比较稳定，不过 etcd 对硬盘 IO 要求较高，因此需要着重关注 IO 相关的指标，生产环境建议至少使用 SSD 的盘做存储。

| 指标 | 解释 |
| ----- | ----- |
| etcd_server_has_leader | etcd 是否有 leader。 |
| etcd_server_leader_changes_seen_total | 切主次数。 |
| etcd_server_proposals_failed_total | 提案失败次数。 |
| etcd_disk_backend_commit_duration_seconds | 提交花费的耗时。 |
| etcd_disk_wal_fsync_duration_seconds | wal 日志同步耗时。 |

## KSM
kube-state-metrics 这个组件，采集的很多指标都只是充当元信息。

| 指标 | 解释 |
| ----- | ----- |
| kube_node_status_condition | Node 节点状态，状态不正常、有磁盘压力等都可以通过这个指标发现。 |
| kube_pod_container_status_last_terminated_reason | 容器停止原因。 |
| kube_pod_container_status_waiting_reason | 容器处于 waiting 状态的原因。 |
| kube_pod_container_status_restarts_total | 容器重启次数。 |
| kube_deployment_spec_replicas | deployment 配置期望的副本数。 |
| kube_deployment_status_replicas_available | deployment 实际可用的副本数。 |
