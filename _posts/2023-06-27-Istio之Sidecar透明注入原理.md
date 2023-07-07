---
layout:     post
title:      Istio之Sidecar透明注入原理
date:       2023-06-27
catalog: true
tags:
    - Istio
---

# Sidecar的注入原理
在kubernetes中，Sidecar容器与应用容器共存于同一个Pod中，在Istio中进行Sidecar注入有两种方式：

1. 通过 istioctl命令行工具手动注入；

2. 通过sidecar-injector自动注入。

这两种方式的最终目的都是在应用的Pod中注入init容器及istio-proxy容器，这两个容器都使用相同的proxyv2镜像。在部署Istio应用时，Sidecar通过configmap配置项 istio-sidecar-injector自动注入。

开启自动注入后，Pod容器的配置片段如下。

（1）istio-proxy容器的配置：

```
# istio-proxy容器的命令行参数
- args:
  ......
  name: istio-proxy
  ......
  securitycontext:
    ......
    # sidecar运行的用户组
    runAsGroup: 1337
    runAsUser: 1337
  # istio-proxy容器挂载的证书及配置文件
  volumeMounts:
  - mountPath : /var/run/secrets/istio
    name: istiod-ca-cert
  - mountPath: /var/lib/istio/data
    name : istio-data
    # Envoy的启动配置文件envoy-rev0.json
  - mountPath: /etc/istio/proxy
    name : istio-envoy
    # 访问Istio用的token
  - mountPath: /var/run/secrets/tokens
    name : istio-token
    # 以文件形式保存的Pod自身服务的名称
  - mountPath: /etc/istio/pod
    name: istio-podinfo
  - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
    name : kube-api-access-rb8w7
    readonly: true
```

runAsGroup和runAsUser指定在运行中使用1337用户的身份，配合iptables规则中的OUTPUT链，判断当前报文是否由Envoy自身发出。

volumeMounts用来将Envoy启动时用到的静态配置文件envoy-rev0.json、Pod身份信息、证书等，以文件形式挂载到Envoy进程可以访问的文件目录下。

（2）istio-init容器的配置：

```
initContainers:
- args:
  ......
  securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    add:
    - NET_ADMIN
    - NET_RAW
    drop:
    - ALL
......
```

istio-init容器负责对Pod配置定制的iptables规则，因此需要被赋予NET_ADMIN和NAT_RAW权限。

* NET_ADMIN：管理网络配置的能力，可以进行网络规则、IP地址、路由表等的管理。

* NET_RAW：使用原始套接字的能力，可以发送和接收IP数据包，也可以进行网络扫描、IP地址伪造等操作。

# 自动注入服务
Sidecar-injector是在Istio中实现自动注入Sidecar的组件，以Kubernetes的Admission Controller（准入控制器）的形式运行。在Istio 1.5版本之后，Sidecar-injector被编译到Istiod进程中。Admission Controller的基本原理是拦截Kube- apiserver的请求，在对象持久化之前、认证鉴权之后进行拦截。

Admission Controller有两种：

1. 内置的;
2. 用户自定义的。

Kubernetes 允许用户以Webhook的方式自定义准入控制器，Sidecar-injector就是这样一种特殊的MutatingAdmissionWebhook。

![](/img/in-post/Istio/Sidecar/sidecar_injector.png)

Sidecar-injector只在创建Pod时进行Sidecar容器注入，在Pod的创建请求到达Kube-apiserver后，首先进行认证鉴权，然后在准入控制阶段，Kube-apisever以REST的方式同步调用Sidecar injector Webook服务进行init容器与istio-proxy容器的注入，最后将Pod对象持久化存储到Eted中。

Sidecar-injector可以通过MutatingWebhookConfiguration API动态配置生效，Istio 1.18中默认的MutatingWebhook "istio-revision-tag-default"配置如下：

```
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: istio-revision-tag-default
......
webhooks:
......
- admissionReviewVersions:
  - v1beta1
  - v1
  clientConfig:
    service:
      name: istiod
      namespace: istio-system
      path: /inject
      port: 443
  failurePolicy: Fail
  matchPolicy: Equivalent
  name: namespace.sidecar-injector.istio.io
  namespaceSelector:
    # 匹配条件
    matchExpressions:
    - key: istio-injection
      operator: In
      values:
        - enabled
......
```


Sidecar-injector对标签匹配“istio-injection:enabled”的命名空间的Pod资源对象的创建生效。Webhook服务的REST访问路径为“/inject”，地址及访问凭证等都在clientConfig字段下进行配置。

Sidecar-injector组件是由Istiod进程实现的，主要用于

1. 维护MutatingWebhookConfiguration；

2. 启动Webhook Server，为应用的工作负载自动注入Sidecar容器。

MutatingWebhookConfiguration对象主要用于监听本地证书的变化及KubernetesMutatingWebhookConfiguration资源的变化，以检查CA证书或者CA数据是否有更新，并且在本地CA证书与MutatingWebhookConfiguration中的CA证书不一致时，自动将本地CA证书更新到MutatingWebhookConfiguration对象中。

# 自动注入流程
Sidecar-injector被编译进Istiod进程，并以轻量级HTTPS服务器的形式处理Kube-apiserver的AdmissionRequest请求。由于Kubernetes Admission Webhook只支持单向认证，所以Sidecar-injector服务器只配置服务器的证书，客户端的Kube-apiserver通过CA校验服务端的证书。

Sidecar-injector在注入时，将根据原始注入模板及默认值生成注入的部分，并插入原始应用容器的YAML配置文件中。原始注入模板位于manifests/charts/istio-control/istio-discovery/files/injection-template.yaml中，详细配置请查看 Istio代码库，其主要包含Sidecar容器模板的定义。原始注入模板及默认ConfigMap资源保存在 istio-system命名空间下，可以根据需要自定义：

```
apiVersion: v1
data:
  # 原始注入模版的上半部分
  config: |-
......
    spec:
      # init-proxy容器的配置
      initContainers:
        args:
        # iptables规则的插入
        - istio-iptables
        - "-p"
......
      containers:
      # Sidecar容器
      - name: istio-proxy
      {{- if contains "/" (annotation .ObjectMeta `sidecar.istio.io/proxyImage` .Values.global.proxy.image) }}
        image: "{{ annotation .ObjectMeta `sidecar.istio.io/proxyImage` .Values.global.proxy.image }}"
      {{- else }}
        image: "{{ .Values.global.hub }}/{{ .Values.global.proxy.image }}:{{ .Values.global.tag }}"
      {{- end }}
        ......
        # Sidecar容器的启动参数
        args:
        - proxy
        - sidecar
        - --domain
......
  # 原始注入模版的下半部分
  values: |-
    {
......
		"proxy": {
		......
		  # 注入的容器名
		  "image": "proxyv2",
		}
}
```

在原始注入模板文件中，上半部分"config"为包含条件判断的模板内容声明，原始注入模板的下半部分"values"为默认值。例如：在config部分对 image 配置引用.Values.global.proxy.image变量占位，对应到values中被替换为"proxyv2"实际的镜像名。

在实际进行注入时，应用的Deployment文件的加载主要通过服务网格配置数据及Pod元数据 ObjectMeta进行。

Pod Sidecar容器的注入过程如下：

![](/img/in-post/Istio/Sidecar/injection_of_pod_sidecar_container.png)

1. 解析Webhook REST请求体，将请求体反序列化为携带AdmissionRequest的AdmissionReview。

2. 解析Pod，将AdmissionRequest中的Object部分反序列化，并匹配注入条件。

3. 利用Pod及服务网格配置组合成的参数对象渲染Sidecar 配置模板并进行后期处理，注入时使用的inject配置文件是使用上面名为“istio-sidecar-injector”的 ConfigMap资源中的内容。

4. 在Webhook的injectPod阶段，将此模板应用到目标Pod上并执行RunTemplate方法，生成并创建目标Pod的YAML配置文件。

5. 利用Pod及渲染后的模板创建JSON patch，通过createPatch生成注入前后的差异部分，构造注入结果的响应AdmissionResponse，在进行JSON编码后，将其发送给HTTP客户端，即Kube-apiserver。

完成后，带有Sidecar自动注入功能的创建Pod的YAML配置文件如下：

```
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubectl.kubernetes.io/default-container: details
    kubectl.kubernetes.io/default-logs-container: details
    prometheus.io/path: /stats/prometheus
    prometheus.io/port: "15020"
    prometheus.io/scrape: "true"
    sidecar.istio.io/status: '{"initContainers":["istio-init"],"containers":["istio-proxy"],"volumes":["workload-socket","credential-socket","workload-certs","istio-envoy","istio-data","istio-podinfo","istio-token","istiod-ca-cert"],"imagePullSecrets":null,"revision":"default"}'
  labels:
    app: details
    pod-template-hash: 5fc8c46b49
    security.istio.io/tlsMode: istio
    service.istio.io/canonical-name: details
    service.istio.io/canonical-revision: v1
......
spec:
  containers:
    - volumeMounts:
      # 作为目录被挂载到istio-proxy容器的/etc/istio/pod目录中
      # 在该目录中包含annotations和1abels文件，这两个文件将被pilot-agent读取
      - mountPath: /etc/istio/pod
        # 引用下面volumes中的istio-podinfo
        name: istio-podinfo
  ......
  volumes:
  - downwardAPI:
      defaultMode: 420
      items:
      - fieldRef:
          apiVersion: v1
          # 获取YAML配置文件中的labels部分
          fieldPath: metadata.labels
        # 文件名
        path: labels
      - fieldRef:
        apiVersion: v1
        # 获取YAML配置文件中的annotations部分
        fieldPath: metadata.annotations
        # 文件名
        path: annotations
    name: istio-podinfo
```

istio-init容器在完成iptables规则配置操作后退出。isito-proxy容器负责启动pilot-agent进程。在容器创建完成后，YAML配置文件中的annotations部分将被挂载到“/etc/istio/pod/labels”文件中，labels部分将被挂载到“/etc/istio/pod/labels”文件中。在pilot-agent进程启动后，可以从这些文件中读取annotations、labels部分，连同在YAML配置文件中设置的环境变量一起作为pilot-agent进程的启动参数。

pilot-agent进程接下来启动Envoy进程。pilot-agent进程在启动后将使用proxyv2镜像目录下的/var/lib/istio/envoy/envoy_bootstrap_tmpl.json文件作为启动模板，在/etc/istio/proxy目录下生成Envoy进程的静态启动文件envoy-rev0.json, 该文件将作为Envoy进程的启动参数-c /etc/istio/proxy/envoy-rev0.json，并监控新启动的Envoy 进程的Stdout、Stderr描述符，当Envoy进程异常退出时，pilot-agent进程也退出，导致整个Pod容器重启。

注意：ConfigMap中的istio-sidecar-injector模板用于生成Pod中的Sidecar容器，而/var/lib/istio/envoy/envoy_bootstrap_tmpl.json模板用于生成Sidecar容器中的Envoy启动配置文件。