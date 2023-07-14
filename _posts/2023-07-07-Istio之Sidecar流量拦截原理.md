---
layout:     post
title:      Istio之Sidecar流量拦截原理
date:       2023-07-07
catalog: true
tags:
    - Istio
---

在完成Sidecar自动注入后，业务在Pod运行期间收发的网络流量将被透明地拦截进Sidecar。其流量拦截基于iptables规则，拦截应用容器的Inbound流量或Outbound流量。Inbound流量简单理解为从Pod外部网口流入进来的流量，比如在一个微服务请求中进入目标服务的请求流量；Outbound流量简单理解为从本Pod内应用向目标微服务发起请求，并最终从本Pod 网口向外部网络发送的流量。目前iptables规则因为不支持UDP转发，所以只设置了拦截TCP流量，会跳过UDP流量。

如下图所示，TCP流量进入Istio应用及从应用发出的流量流出Pod的过程，其中①~③表示Inbound流量，④~⑥表示Outbound流量。

![](/img/in-post/Istio/sidecar/istio_traffic_direction.png)

①Inbound流量在进入Pod的网络协议栈时首先被iptables规则拦截。

②iptables规则将报文转发给Envoy。

③Envoy根据自身监听器的配置，将流量转发给应用进程。注意，Envoy在将Inbound流量转发给应用时，将根据匹配条件跳过iptables规则的拦截，将流量发送到后端服务。

④后端服务的主动请求流量被 iptables规则拦截。

⑤iptables规则将后端服务响应的流量转发给Envoy。

⑥Envoy根据自身配置决定是否将流量转发到容器外。

# Sidecar Redirect模式
Istio Sidecar代理的默认流量拦截模式，适用于容器场景中。Sidecar与用户进程共享同一个网络命名空间，工作在相同的网络协议栈上，Sidecar对协议栈iptables规则的配置，将影响用户应用程序报文的流向，可以透明地拦截用户报文并进行七层处理。

注意，Envoy在接收下游接收客户端的连接后，需要将处理后的请求发送到新创建的上游连接，新连接的目标为Inbound端的Envoy或目标服务。在Sidecar Redirect模式下，新建的上游连接没有指定绑定源地址为客户端的地址，因此在经过本Pod网络空间的iptables内核协议栈后，将根据报文的目标地址及路由表关联的网络设备子网信息，分配一个随机端口作为新连接的源地址及源端口。因此对于目标接收端来说，相当于屏蔽了真正的原始客户端的请求地址，可能使得之前根据四层连接的源地址判断如何提供服务的业务逻辑无法继续使用。Sidecar Redirect模式可以通过以下默认方式配置：

```
template:
  metadata:
    annotations: sidecar.istio.io/interceptionMode: REDIRECT
```

首先根据Pod内的Envoy进程Pid通过nsenter命令进入istio-proxy容器的网络空间：

```
nsenter -n -t PID bash
```

进入istio-proxy容器网络空间后使用iptables命令观察当前配置的iptables规则：
```
# iptables -t nat -S

-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N ISTIO_INBOUND
-N ISTIO_IN_REDIRECT
-N ISTIO_OUTPUT
-N ISTIO_REDIRECT
-A PREROUTING -p tcp -j ISTIO_INBOUND
-A OUTPUT -p tcp -j ISTIO_OUTPUT
-A ISTIO_INBOUND -p tcp -m tcp --dport 15008 -j RETURN # Pn1
-A ISTIO_INBOUND -p tcp -m tcp --dport 15090 -j RETURN
-A ISTIO_INBOUND -p tcp -m tcp --dport 15021 -j RETURN
-A ISTIO_INBOUND -p tcp -m tcp --dport 15020 -j RETURN
-A ISTIO_INBOUND -p tcp -j ISTIO_IN_REDIRECT # Pn2
-A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-ports 15006
-A ISTIO_OUTPUT -s 127.0.0.6/32 -o lo -j RETURN # On1
-A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -m owner --uid-owner 1337 -j ISTIO_IN_REDIRECT # On2
-A ISTIO_OUTPUT -o lo -m owner ! --uid-owner 1337 -j RETURN # On3
-A ISTIO_OUTPUT-mowner --uid-owner 1337 -j RETURN # On4
-A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -m owner --gid-owner 1337 -j ISTIO_IN_REDIRECT # On2
-A ISTIO_OUTPUT -o lo -m owner ! --gid-owner 1337 -j RETURN # On3
-A ISTIO_OUTPUT -m owner --gid-owner 1337 -j RETURN # On4
-A ISTIO_OUTPUT -d 127.0.0.1/32 -j RETURN # On5
-A ISTIO_OUTPUT -j ISTIO_REDIRECT # On6
-A ISTIO_REDIRECT -p tcp -j REDIRECT --to-ports 15001
```

将iptables规则分为三种场景：Outbound、Inbound、Outbound+Inbound。

以iptables链名+iptables表名+规则编号的缩写形式来表示标记规则，例如：On1表示OUTPUT链中Nat表的第1条标记规则，Pn1表示PREROUTING链中Nat表的第1条标记规则，以此类推。

报文在经过每个过滤器链时都将根据优先级的不同，进入不同的iptables表进行处理，例如对于PREROUTING链，依次经过Raw表、conntrack表、Mangle表、Nat表，其中Mangle表不论当前连接在conntrack表内记录的状态如何，都会对每个报文进行处理；而Nat表会判断当前连接在conntrack表内的连接记录是否处于Established状态，如果处于Established状态，则跳过Nat表。因此只在连接握手时处理经过Nat表的报文，在连接建立后，本连接的数据报文将不再进入Nat表进行处理。

Envoy与Pod内的pilot-agent进程采用了UDS（Unix Domain Socket）协议通信，该协议不属于三层协议，不受iptables规则的限制。

Redirect模式下的典型iptables规则

| 匹配规则 | 功能描述 |
|----|-------|
| -P PREROUTING ACCEPT | 接收进入PREROUTING链的报文 |
| -N ISTIO_INBOUND | 声明一个自定义的链ISTIO_INBOUND，可以使用-A对此链添加过滤规则 |
| -A PREROUTING -p tcp -j ISTIO_INBOUND | 将进入PREROUTING链的TCP流量跳转到ISTIO_INBOUND链做进一步处理 |
| -A ISTIO_INBOUND -p tcp -m tcp --dport 15008 -j RETURN | 对进入ISTIO_INBOUND链的目标端口为15008的TCP流量不做特殊处理，直接让其通过 |
| -A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-ports 15006 | 对进入ISTIO_IN_REDIRECT链的TCP流量进行报文修改，REDIRECT对应DNAT修改方式，修改目标端口为15006 |
| -A ISTIO_OUTPUT -s 127.0.0.6/32 -o lo -j RETURN | 对进入ISTIO_OUTPUT链的源地址为127.0.0.6的报文且目标网络设备为lo本地设备的流量，不进行特殊处理 |
| -A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -m owner --uid-owner 1337 -j ISTIO_IN_REDIRECT | 对进入ISTIO_OUTPUT链且目标地址虽然不为127.0.0.1但判断目标网络设备为本地（即Pod自身地址）的报文，若报文发送进程为uid=1337，则Envoy自身转到ISTIO_IN_REDIRECT链继续处理 |

## Rediret模式下Outbound场景的访问流程
Istio Outbound为东西向流量中主动向目标服务发起访问的请求端。

![](/img/in-post/Istio/sidecar/redirect_outbound.png)

| 序号 | 应用 | 方向 | 地址说明 | 处理链/表 | 匹配规则 | 功能描述 |
|-----|-----|----|------|------------|--------|------------------|
| 1 | frontend | 发送 | src:ip1 sport:port1，dst:forecast dport:port2 | OUTPUT链/Nat表 | -A ISTIO_OUTPUT -j ISTIO_REDIRECT # On6 <br> -A ISTIO_REDIRECT -p tcp_-j REDIRECT --to-ports 15001 | 当frontend 应用访问forecast 服务时，connect系统函数经过DNS解析后得到forecast的ClusterIp地址并发送SYN报文，SYN报文被iptables规则拦截，并由OUTPUT链通过DNAT方式修改目标地址为Envoy ip1+15001端口，连接属性sockopt保留原始目标服务地址ClusterIp及端口 |
| 2 | Envoy | 接收 | dst:ip1 dport:15001 | 无 | 无 | 在Envoy配置文件中指定original_dst作为监听插件，当用户的连接到达时，在插件内执行getsockopt系统调用，还原连接的原始目标服务地址Clusterlp及端口。随后，在经过Envoy内负载均衡策略的处理后，得到上游连接的目标实例地址并创建连接，此时目标实例为forecast的Pod容器 |
| 3 | Envoy | 发送 | src:ip1 sport:port3，dst:ip2 dport:port2 | OUTPUT链/Nat表 | -A ISTIO_OUTPUT -m owner --gid-owner 1337 -j RETURN # On4 | 结合#On4条件，这里指Envoy向forecast发送的流量不再被拦截 |
| 4 | Envoy | 接收 | src:ip2 sport:port2，dst:ip1 dport:port3 | PREROUTING表 | | 目标服务forecast Pod接收请求 |
| 5 | Envoy | 发送 | src 127.0.0.1 sport:15001 dst:ip1 dport:port1 | 无 |  | 由于前面DNAT的缘故，Envoy返回给请求端frontend应用的报文，其源地址将被修改为127.0.0.1:15001 |

表中的第2、5项，由于DNAT的参与，整个连接从发起方与接收方看到的源地址、源端口与目标地址、目标端口是不同的，可以通过conntrack -L命令查看，例如：

```
tcp 6 117 ESTABLISHED src=10.244.109.55 dst=10.111.181.176 sport=42228 dport=8123 src=127.0.0.1 dat=10.244.109.55 sport=15001 dport=42228 [ASSURED] mark=0 use=1
tcp 6 431932 ESTABLISHED src=10.244.109.55 dst=10.244.109.53 sport=47930 dport=8123 src=10.244.109.53 dst=10.244.109.55 sport=8123 dport=47930 [ASSURED] mark=0 use=1
```

上面一条记录反映的是frontend应用与Envoy建立的下游连接，下面一条为Envoy创建的与forecast目标Pod的上游连接。其中10.244.109.55为frontend的Pod地址，forcast地址为10.111.181.176，forcast的Pod实例地址为10.244.109.53。从下游连接对应的记录可以看出，前半部分src与dst为从forecast发起连接时的五元组信息；后半部分为从Envoy发送并建立连接后的反方向的五元组信息，它们的地址可以是不对称的。这是因为Envoy在将SYN ACK包发送到OUTPUT链前，在路由阶段发现可以通过本地lo设备将报文发送回frontend 应用，而选择了127.0.0.1作为源地址。

## Redirect模式下Pod内场景的访问流程
有时需要在Istio的Pod中使用curl等命令测试本Pod内目标服务的连通性或直接访问Envoy的管理端口获取运行状态，在这种场景中不希望报文被iptables规则拦截而进入Envoy，所以将根据目标地址、目标网络设备及是否为Envoy容器发出连接作为条件进行判断。如果请求端的应用使用环回地址127.0.0.1作为目标地址或者本地网络设备lo，则不再进行拦截。

![](/img/in-post/Istio/sidecar/redirect_pod.png)

| 序号 | 应用 | 方向 | 地址说明 | 处理链/表 | 匹配规则 | 功能描述 |
|-----|-----|----|------|------------|--------|------------------|
| 1 | curl | 发送 | dst:127.0.0.1 | OUTPUT链/Nat表 | -A ISTIO_OUTPUT -d 127.0.0.1/32 -j RETURN # On5 | 进入 Pod网络空间后，执行 curl访问 Envoy管理端不被拦截，直接访问 |
| 2 | curl | 发送 | dst:lo | OUTPUT链/Nat表 | -A ISTIO_OUTPUT -o lo -m owner ! --uid-owner 1337 -j RETURN # On3 | 进入 Pod 网络空间后，执行 curl通过localhost地址访问forecast不被拦截，直接访问 |

## Redirect模式下Inbound场景的访问流程
在Istio服务网格内，Inbound流量指从Pod外进入Pod内的流量。比如frontend应用在访问forecast时，应用数据报文进入forecast Pod时被拦截进入Envoy 15006监听端口的流量，而一些外部的控制流量将直接访问Envoy的管理端口，比如15008、15090等，这些流量不应被拦截。

![](/img/in-post/Istio/sidecar/redirect_inbound.png)

| 序号 | 应用 | 方向 | 地址说明 | 处理链/表 | 匹配规则 | 功能描述 |
|-----|-----|----|------|------------|--------|------------------|
| 1 | frontend | 接收 | src:ip1 sport:port1，dst:ip2 dport:port2 | PREROUTING链/Nat表 | -A ISTIO_INBOUND -p tcp -j ISTIO_IN_REDIRECT # Pn2 <br> -A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-ports 15006 | 当frontend容器访问forecast的Pod地址ip2时，报文被iptables规则拦截并通过DNAT方式修改目标地址为Envoy ip1+15006端口，且在报文中保留原始目标地址及端口。此原始目标地址及端口可以在VirtualInbound监听器内通过getsockopt系统调用获取。与前面Outbound流程中目标地址为forecast不同，这里目标地址dst为本Pod的地址ip2 |
| 2 | Envoy | 接收 | src:ip1 sport:port2，dst:ip2 dport:15008/15090/15020/15020 | PREROUTING链/Nat表 | -A ISTIO_INBOUND -p tcp -m tcp --dport 15008 -j RETURN # Pn1 <br> -A ISTIO_INBOUND -p tcp -m tcp --dport 15090 -j RETURN <br> -A ISTIO_INBOUND -p tcp -m tcp --dport 15021 -j RETURN <br> -A ISTIO_INBOUND -p tcp -m tcp --dport 15020 -j RETURN | 当Pod外部的监控系统通过15090拉取Envoy记录的可观测性数据时，请求不应被拦截，而是由Envoy监听器处理 |
| 3 | Envoy | 发送 | src:127.0.0.1 sport:port3，dst:127.0.0.1 dport:port2 | OUTPUT链/Nat表 | -A ISTIO_OUTPUT -m owner --gid-owner 1337 -j RETURN # On4 | 这里指Envoy发送到本Pod后端forecast的流量不再被拦截 |
| 4 | Envoy | 发送 | src:127.0.0.1 sport:15006，dst:ip1 dport:port1 | 无 | | 由于前面DNAT的缘故，Envoy返回给发送端frontend 的Pod的报文，其源地址将被修改为127.0.0.1:15006 |
| 5 | Envoy | 路由 | 无 | 无 |  | 当Inbound请求没有根据目标地址找到后端服务时，下游请求将被转发到Envoy内置PassthroughClusterlpv4服务，此服务创建的上游连接绑定127.0.0.6作为源地址，并按原始目标地址转发下游请求 |
| 6 | Envoy | 发送 | src 127.0.0.6 | OUTPUT链/Nat表 | -A ISTIO_OUTPUT -s 127.0.0.6/32 -o lo -j RETURN # On1 | PassthroughClusterlpv4配置了固定的源地址127.0.0.6，用于与Envoy内可以匹配的Cluster后端服务访问场景On4进行区分，此时在OUTPUT链处理阶段，被转发的请求将不再被拦截。同时由于目标网络设备为lo，因此可以访问到本Pod内未注册的后端服务 |

## Redirect模式下Outbound +lnbound（自身环回访问）场景的访问流程
在Redirect模式下，当容器中同时存在服务访问者和提供者，且在经过Sidecar负载均衡后又恰巧选择了本Pod内的服务作为目标地址时，就被称为Outbound+Inbound场景。

![](/img/in-post/Istio/sidecar/redirect_outbound_inbound.png)

| 序号 | 应用 | 方向 | 地址说明 | 处理链/表 | 匹配规则 | 功能描述 |
|-----|-----|----|------|------------|--------|------------------|
| 1 | frontend | 发送 | src:ip1 sport:port1，dst:forecast dport:port2 | PREROUTING链/Nat表 | -A ISTIO_OUTPUT -j ISTIO_REDIRECT # On6 <br> -A ISTIO_REDIRECT -p tcp -j REDIRECT --to-ports 15001 | Pod内的frontend容器访问forecast，经过DNS解析后得到Clusterlp地址，连接请求被iptables规则拦截，iptables规则中的REDIRECT命令通过DNAT方式修改报文目标地址为Envoy ip1+15001端口，并在报文内保留原始目标服务地址ClusterIp及端口 |
| 2 | Envoy | 发送 | src:ip1 sport:port3，dst:ip1 dport:port2 | OUTPUT链/Nat表 | -A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -m owner --uid-owner 1337 -j ISTIO_IN_REDIRECT # On2 | Envoy在收到客户端应用的连接后，恢复原始目标服务地址Clusterlp及端口，经过负载均衡后选择的目标实例恰好为本地址及端口Pod内forecast容器的 ip1地址及端口port2。 <br> 在Envoy创建到目标地址ip1的上游连接时，iptables规则匹配#On2条件，使用ISTIO_IN_REDIRECT子链中的REDIRECT命令将报文目标地址修改为Envoy自身及Inbound端口15006。其效果与从Pod外部进入的Inbound流量在 PREROUTING链中时的处理一致，之后的请求将进入Envoy的15006端口的监听器 |
| 3 | Envoy | 发送 | src:127.0.0.1 sport:port4，dst:127.0.0.1 dport:port2 | OUTPUT链/Nat表 | -A ISTIO_OUTPUT -m owner --gid-owner 1337 -j RETURN # On4 | 同前面Inbound场景中对后端服务的访问流程一致；Envoy发送到本Pod后端forecast的流量不再被拦截 |
| 4 | Envoy | 发送 | src:127.0.0.1 sport:15006，dst:ip1 dport:port3 | 无 |  | Envoy Inbound完成处理后返回给Envoy Outbound的报文，由于前面DNAT的缘故，响应报文的源地址src被修改为127.0.0.1:15006 |
| 5 | Envoy | 发送 | src:127.0.0.1 sport:15001，dst:ip1 dport:port1 | 无 |  | Envoy Outbound 完成处理后返回给frontend容器的报文，由于DNAT的缘故，响应报文的源地址src被修改为127.0.0.1:15001 |

在Sidecar的默认模式下，Envoy作为用户空间的四层或七层代理，担负着将Outbound方向的Kubernetes服务地址负载均衡后确定Pod地址的过程，而对于Inbound方向不需要再处理虚拟服务的地址，而是将对目标服务地址的访问转成一个目标地址为127.0.0.1的新连接的访问。

在这种模式下，在每个方向都需要创建新的Socket连接，虽然可以配置新连接的创建数量（连接池），但还是较大增加了文件描述符fd的消耗。

另外需要注意的是，前面提到tcpdump工作在设备二层上，因此对于Inbound方向PREROUTING阶段所做的DNAT，无法使用15006端口作为抓包过滤参数，但可以通过conntrack -L命令查询当前网络连接状态并过滤包含端口15001或15006的连接信息。

Istio默认使用Redirect模式，当服务端需要根据客户端的源地址进行业务判断时，可以通过应用协议携带客户端源地址的方式实现，比如在HTTP头部添加“x-forward-for”。但对于服务端需要基于四层源地址判断的场景，则很难实现，在这些场景中需要使用Tproxy模式，此模式比Rediret模式复杂。

# Sidecar Tproxy模式
待补充。

# Ingress网关模式	
Ingress网关常常被部署为运行于用户空间的独立进程，通过统一的对外服务端口接收外部的流量请求，经过四层或七层处理后，作为客户端代理创建与服务的连接后转发客户端的请求数据，反过来将服务端的响应转发给客户端。在这个过程中将创建新的socket，因此会比较消耗节点的文件描述符数量，但同时增加了灵活性。此时网关节点不需要配置特殊的iptables规则，因为客户端需要明确连接网关监听地址，无须进行透明流量拦截。

Ingress网关 Pod在启动时已经默认配置了针对不同协议的路由端口及nodeport：
```
kubectl get service istio-ingressgateway -n istio-system-o jsonpath='{.spec.ports}'

[{"name":"status-port","nodePort":31191,"port":15021,"protocol":"TCP","targetPort":15021},{"name":"http2","nodePort":31631,"port":80,"protocol":"TCP","targetPort":8080},{"name":"https","nodePort":30995,"port":443,"protocol":"TCP","targetPort":8443}]
```

获取当前Istio Ingress网关服务地址的代码：
```
kubectl get service istio-ingressgateway -n istio-system -o wide

istio-system istio-ingressgateway LoadBalancer 10.99.58.16
```

Ingress网关默认不代理任何服务，因此不启动监听。举例来说，如果配置了网关及VirutalService，则网关的配置如下：

```
apiversion: networking.istio.io/vlbeta1
kind: Gateway
metadata:
  name: demo-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  apps:
  - port:
      # 此端口为前面约定的路由端口号，需要针对不同的协议进行选择
      number: 80
      name: http
      protocol:HTTP
    hosts:
    # domain域名，在用户请求时根据Host内容进行匹配
    - "wistio.cc"
```

接下来创建VirtualService配置，指明路由的端口的HTTP中hosts域的内容相关联，当用户请求Ingress网关的80端口并在HTTP头部携带“Host: wistio.cc”时，请求将被实际映射到Istio服务：

```
apiversion: networking.istio.io/vlbeta1
kind: VirtualService
metadata:
  name: demo-http-route
spec:
  hosts:
  # 匹配gateway中的hosts域名
  - "wistio.cc"
  gateways:
  #匹配gateway
  - demo-gateway
  http:
  # 匹配协议支持的路由规则
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        port:
          number: 8123
        # 目标服务名，之后会根据服务的负载均衡策略来选择Endpoint
        host: forecast
```

VirtualService用于通过gateways值匹配前面的路由项，并提供hosts匹配HTTP请求中的头部domain信息，并最终将请求转发到后端的destination。以上两部分将自动转换为如下Ingress网关内的iptables Nat表规则及Envoy配置文件。可以用节点地址localhost:nodePort的形式访问网关代理的服务，targetPort为Ingress网关Pod内的Envoy进程启动的监听地址。

节点与Ingress网关相关的iptables Nat表规则如下：

```
iptables -t nat -S | grep KUBE-SVC-G6D3V5KS3PXPUEDS

-A KUBE-SERVICES -d 10.99.58.16/32 -p tcp -m comment --comment "istio-system/istio-ingressgateway:http2 cluster IP" -m tcp --dport 80 -j KUBE-SVC-G6D3V5KS3PXPUEDS
-A KUBE-NODEPORTS -p tcp -m comment --comment "istio-system/istio-ingressgateway:http2" -m tcp --dport 31631 -j KUBE-SVC-G6D3V5KS3PXPUEDS
-A KUBE-SVC-G6D3V5KS3PXPUEDS -m comment --comment "istio-system/istio-ingressgateway:http2" -j KUBE-SEP-X2TXEPVKEONDIKYY
-A KUBE-SEP-X2TXEPVKEONDIKYY -p tcp -m comment --comment "istio-system/istio-ingressgateway:http2" -m tcp -j DNAT --to-destination 10.244.145.72:8080
```

访问目标服务istio-ingressgateway，其地址为10.99.58.16:80时，或在访问本节点地址:31631时，都会跳转到KUBE-SVC-G6D3V5KS3PXPUEDS链。80及31631端口并没有进行真正监听，而是作为iptables转发匹配规则。此链在经过KUBE-SEP-X2TXEPVKEONDIKYY链内的DNAT地址转换后，将报文地址重定向到Ingress网关Pod 10.244.145.72的8080监听端口。

观察Ingress网关Pod的配置文件：

```
kubectl exec -it istio-ingressgateway-76df49f94d-x1jqt -n istio-system -- curl http://127.0.0.1:15000/config_dump

"dynamic_listeners": [
{
 "name": "0.0.0.0_8080",
 ......
   "filter_chains":
   {
    "filters":[
    {
     "name": "envoy.filters.network.http_connection_manager",
     "typed_config": {
       "route_config_name": "http.8080"
```

从配置可以看出，Ingress网关实际监听在动态加入的8080端口上，并使用名为http.8080的路由，路由配置如下：

```
"dynamic_route_configs": [
{
 "name":"http.8080",
 "virtual_hosts": [
 {
  "name": "wistio.cc:80",
  "domains": [
    "wistio.cc",
    "wistio.cc:*"
  ],
  routes": [
  {
   "match" {
     "prefix": "/",
     "case_sensitive": true
   },
   "route": {
     "cluster": "outbound|81231|forecast.default.svc.cluster.local",
```

http.8080路由在处理用户请求时，如果请求匹配HTTP头部的domain，则此请求将被转发到VirtualService定义的服务名outbound|81231|forecast.default.svc.cluster.local。

![](/img/in-post/Istio/sidecar/ingress_gateway.png)

需要注意的是，如果后端服务需要根据四层源地址信息作为判断依据，那么在Ingress网关创建新连接时，需要设置IP_TRANSPARENT标志，并将客户端地址绑定为新连接的本地地址。同时，服务端也要设置默认的路由规则为连接网关的二层设备，并指定默认路由via为网关地址。将使后端服务在服务网格中的部署不透明，因此Istio不支持这种透明的Ingress网关模式。在实际应用中，服务端通过解析HTTP协议头X-Forward-for来获得客户端的地址。