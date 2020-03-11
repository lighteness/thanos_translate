# 使用Prometheus、Thanos监控Kubernetes集群


## 介绍

恭喜你！当你阅读这篇文章的时候，我相信你一定已经说服了你的经理，或者是公司CTO，选择容器和Kubernetes作为微服务治理平台，去转型升级你们公司的软件产品。

你非常非常的happy，一起都按照计划进行，你创建了你的第一个Kubernetes集群（三大主流云服务提供商，微软云Azure，亚马逊云AWS和谷歌云GCP都提供了非常方便的方式部署Kubernetes平台），你开发了你的第一个容器化应用，然后把它部署到了你的Kubernetes集群上。看上去容易极了，真的是这样吗？:)

过了一段时间后，你开始意识到，事情开始变得有一些复杂，你有多个应用需要部署到集群上，因此你需要有一个Ingress Controller，再然后，在投产之前，你开始想知道你的应用的性能如何，你又开始寻找监控解决方案，幸运的是，你发现了[Prometheus](https://prometheus.io/)，部署后，再加上[Grafana](https://grafana.com/)，完美！

后来，你开始好奇——我的Prometheus实例怎么就只有一个副本？要是哪天挂了，该怎么办？要是我的Prometheus要版本升级，该怎么办？我的Prometheus可以保存我的监控数据多久呀？要是整个集群包括Prometheus都完蛋了，我该怎么办？难道我要建另一个集群来满足高可用或者灾备？面对多个Prometheus实例，我该如何才能有个集中化的视图？

好吧，不卖关子了，一些聪明蛋已经把这些问题解决了。

## 典型的Kubernetes集群

下面的一张图，阐明了在Kubernetes环境中，一般情况下的部署结构——

。。图片1


部署结构包含三层：

1. 底层的虚拟机——master节点和worker节点 
2. kubernetes基础架构
3. 上层的用户应用 

不同的组件之间通常通过HTTP(s)（REST或者gRPC）相互通信，其中一些组件通过Ingress把APIs暴露到集群外部，这些APIs主要用途是——

1. 通过Kubernetes API Server对集群管理
2. 通过Ingress Controller暴露应用服务

在某些场景下，应用可以通过如Egress的方式把流量发往集群外部来消费如Azure SQL，Azure Blob或其他第三方服务。


## 要监控些什么？
想要监控Kubernetes，就应该把上面提及的三个更层次都应该考虑进去。

**底层的虚拟机**：要确保底层虚拟机是健康的，下面几个监控参数要收集——
- 节点数量
- 每个节点的资源使用率（CPU，内存，磁盘，网络带宽）
- 节点状态（就绪，未就绪，等其他状态）
- 每个节点上运行的pod数量

**Kubernetes基础架构**：要确保Kubernetes基础架构是健康的，下面几个监控参数要收集——
- Pods健康 — ready, status, restarts, age
- Deployments状态 — desired, current, up-to-date, available, age
- StatefulSets状态
- CronJobs执行相关统计数据
- Pod资源利用率(CPU和内存)
- 健康检查
- Kubernetes Events事件
- API Server请求相关统计数据
- Etcd统计数据
- 挂载volumes统计数据

**用户应用**：每一个应用应该依据自己的核心功能，暴露自己的监控数据，然而，存在一些常见的统计数据，比如：
- HTTP请求(总请求数， 总请延迟， 响应码等)
- 面向依赖外部服务的请求连接数(例如数据库请求连接数)
- 线程数

收集上述统计数据，会使得你构建起有价值有意义的**告警**和**监控面板**，我们马上就会讲到这些内容。

## Thanos
[Thanos](https://github.com/thanos-io/thanos)面对上述抛出的问题，可以提供**高可用**解决方案，并且有着**不受限制的数据存储能力**，无缝衔接Prometheus。Thanos是一个开源项目，其内部由多个组件组成。

Thanos使用Prometheus存储格式，把历史数据以相对高性价比的方式保存在对象存储里，同时兼有较快的查询速度。此外，它能还对你所有的Prometheus提供**全局查询视图**。

Thanos主要组件有：
- 边车组件（Sidecar）：连接Prometheus，并把Prometheus暴露给查询网关（Query Gateway），以供实时查询，并且可以上传Prometheus数据给云存储，以供长期保存。
- 查询网关（Query Gateway）：实现了Prometheus API，与其他组件（如边车组件Sidecar，或是存储网关Store Gateway）一起协同工作
- 存储网关（Store Gateway）：将云存储中的数据内容暴露出来
- 压缩器（Compactor）：将云存储中的数据进行压缩和下采样
- 接收器（Receiver）：从Prometheus’ remote-write WAL（Prometheus远程预写式日志）获取数据，暴露出去或者上传到云存储
- 规则组件（Ruler）：针对数据进行评估和报警

在这边文章里，我们主要谈谈前三个组件。


。。。图

## 部署Thanos

我们将开始把Thanos边车组件（Sidecar）部署到我们的Kubernetes集群。在这个集群，我们已经放置了我们的应用，Prometheus和Grafana。

虽然有很多种方式安装Prometheus，我更喜欢[Prometheus-Operator](https://github.com/coreos/prometheus-operator)这种方式，它能让部署，管理和定义Prometheus更加容易。

安装Prometheus-Operator最容易的方式就是使用[Helm chart](https://github.com/helm/charts/tree/master/stable/prometheus-operator)，提供了对高可用的支持，Thanos边车组件（Sidecar）的注入，以及监控虚拟机、监控kubernetes基础架构、监控你的应用所需的预制报警。

在部署Thanos边车组件（Sidecar）之前，我们需要一个Kubernetes Secret，里面放置了如何连接云存储所需的详细信息，在这个demo中，我将使用微软云Azure。

创建一个快存储账号——

```
az storage account create ——name <storage_name> ——resource-group <resource_group> ——location <location> ——sku Standard_LRS ——encryption blob
```

然后，创建一个文件夹（在azure存储概念上，称作container更准确）
```
az storage container create ——account-name <storage_name> ——name thanos
```

获取存储秘钥——
```
az storage account keys list -g <resource_group> -n <storage_name>

```

为存储配置创建一个yaml文件 (thanos-storage-config.yaml)——
```
type: AZURE
config:
  storage_account: "<storage_name>"
  storage_account_key: "<key>"
  container: "thanos"
```

基于此yaml文件，转化成一个Kubernetes Secret —
```
kubectl -n monitoring create secret generic thanos-objstore-config ——from-file=thanos.yaml=thanos-storage-config.yaml
```


创建另一个yaml文件(prometheus-operator-values.yaml)来覆盖默认的Prometheus-Operator配置——
```
prometheus:
  prometheusSpec:
    replicas: 2      # work in High-Availability mode
    retention: 12h   # we only need a few hours of retenion, since the rest is uploaded to blob
    image:
      tag: v2.8.0    # use a specific version of Prometheus
    externalLabels:  # a cool way to add default labels to all metrics 
      geo: us          
      region: eastus
    serviceMonitorNamespaceSelector:  # allows the operator to find target config from multiple namespaces
      any: true
    thanos:         # add Thanos Sidecar
      tag: v0.3.1   # a specific version of Thanos
      objectStorageConfig: # blob storage configuration to upload metrics 
        key: thanos.yaml
        name: thanos-objstore-config
grafana:           # (optional) we don't need Grafana in all clusters
  enabled: false
```

然后部署：
```
helm install ——namespace monitoring ——name prometheus-operator stable/prometheus-operator -f prometheus-operator-values.yaml
```

现在你应该拥有了一个**高可用**的Prometheus运行在你的集群中，同时Thanos边车组件（Sidecar）已经能够将你的监控数据上传至Azure块存储对象，并且存储容量可以没有上限。

为了让存储网关（Store Gateway）可以访问这些Thanos边车组件（Sidecar），我们将需要通过一个Ingress把他们暴露出去，这里我使用[Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx)，但是你可以使用其他的Ingress Controller，只要能支持gRPC协议（[Envoy](https://www.envoyproxy.io/)可能是最佳的选择）。

为了使存储网关（Store Gateway）和边车组件（Sidecar）之间的通信安全，我们使用TLS双向认证技术（mutual-TLS），这意味着客户端要验证服务端，服务端也要验证客户端。

假设你已经有了.pfx文件你可以使用openssl来抽取私钥，公钥和CA——
```
# 公钥
openssl pkcs12 -in cert.pfx -nocerts -nodes | sed -ne '/-BEGIN PRIVATE KEY-/,/-END PRIVATE KEY-/p' > cert.key
# 私钥
openssl pkcs12 -in cert.pfx -clcerts -nokeys | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > cert.cer
# CA
openssl pkcs12 -in cert.pfx -cacerts -nokeys -chain | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > cacerts.cer
```

以此基础上，创建2个Kubernetes Secrets
```
# a secret to be used for TLS termination
kubectl create secret tls -n monitoring thanos-ingress-secret ——key ./cert.key ——cert ./cert.cer
# a secret to be used for client authenticating using the same CA
kubectl create secret generic -n monitoring thanos-ca-secret ——from-file=ca.crt=./cacerts.cer
```

确保你有一个域（domain）用来解析你的Kubernetes机器，并且创建2个子域（sub-domain），用来访问各个Thanos边车组件（Sidecar）
```
thanos-0.your.domain
thanos-1.your.domain
```

现在我们来创建几个Ingress规则（你需要用你自己的host值替代这里的host）
```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: prometheus
  name: thanos-sidecar-0
spec:
  ports:
    - port: 10901
      protocol: TCP
      targetPort: grpc
      name: grpc
  selector:
    statefulset.kubernetes.io/pod-name: prometheus-prometheus-operator-prometheus-0
  type: ClusterIP
——-
apiVersion: v1
kind: Service
metadata:
  labels:
    app: prometheus
  name: thanos-sidecar-1
spec:
  ports:
    - port: 10901
      protocol: TCP
      targetPort: grpc
      name: grpc
  selector:
    statefulset.kubernetes.io/pod-name: prometheus-prometheus-operator-prometheus-1
  type: ClusterIP
——-
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-secret: "monitoring/thanos-ca-secret"
  labels:
    app: prometheus
  name: thanos-sidecar-0
spec:
  rules:
  - host: thanos-0.your.domain
    http:
      paths:
      - backend:
          serviceName: thanos-sidecar-0
          servicePort: grpc
  tls:
  - hosts:
    - thanos-0.your.domain
    secretName: thanos-ingress-secret
——-
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-secret: "monitoring/thanos-ca-secret"
  labels:
    app: prometheus
  name: thanos-sidecar-1
spec:
  rules:
  - host: thanos-1.your.domain
    http:
      paths:
      - backend:
          serviceName: thanos-sidecar-1
          servicePort: grpc
  tls:
  - hosts:
    - thanos-1.your.domain
    secretName: thanos-ingress-secret
```    

现在我们建立起了一条安全方式，让我们可以从集群外部访问我们的Thanos边车组件（Sidecar）。


### Thanos 集群
从上面的Thanos例图中，你看到我选择把Thanos部署在一个单独的集群里，因为我希望有一个专属的集群给Thanos，这样如果有必要，我可以很方便的重建，同时从安全角度，其他的工程师不需要介入真正的生产环境，也能访问Thanos。

为了部署Thanos组件，我选择了使用这个[Helm chart](https://github.com/arthur-c/thanos-helm-chart)（并非官方版本）

创建一个yaml文件thanos-values.yaml来覆盖默认的chart设置——
```

# Thanos query configuration
query:
  replicaCount: 1
  logLevel: debug
  queryReplicaLabel: prometheus_replica
  stores:
    - thanos-store-grpc:10901
    - thanos-0.your.domain:443
    - thanos-1.your.domain:443
  tlsClient:
    enabled: true

objectStorageConfig:
  enabled: true

store:
  tlsServer:
    enabled: true
```

因为存储网关（Store Gateway）需要从我们之前创建的块存储中读取数据，我们也需要基于我们之前创建的thanos-storage-config.yaml创建一份kubernetes secret。
```
kubectl -n thanos create secret generic thanos-objstore-config ——from-file=thanos.yaml=thanos-storage-config.yaml
```

为了部署这个chart，我们讲用到之前我们早些时候创建的证书，把他们当做值注入进去。
```
helm install ——name thanos ——namespace thanos ./thanos -f thanos-values.yaml ——set-file query.tlsClient.cert=cert.cer ——set-file query.tlsClient.key=cert.key ——set-file query.tlsClient.ca=cacerts.cer ——set-file store.tlsServer.cert=cert.cer ——set-file store.tlsServer.key=cert.key ——set-file store.tlsServer.ca=cacerts.cer
```
这将同时安装上Thanos查询网关（Query Gateway）和Thanos存储网关（Store Gateway），并在他们之间建立起一条安全通道。


### 验证
为了验证一切工作正常，你可以对Thanos查询网关（Query Gateway）HTTP 服务使用 port-forward 来转发 ——-
```
kubectl -n thanos port-forward svc/thanos-query-http 8080:10902
```

打开你的浏览器，输入 http://localhost:8080 你将看到Thanos UI!——

。。。图


### Grafana
为了安装上Grafana，你可以用Grafana的Helm chart。

创建一份yaml文件grafana-values.yaml，内容如下——

```
datasources:
 datasources.yaml:
   apiVersion: 1
   datasources:
   - name: Prometheus
     type: prometheus
     url: http://thanos-query-http:10902
     access: proxy
     isDefault: true
dashboardProviders:
 dashboardproviders.yaml:
   apiVersion: 1
   providers:
   - name: 'default'
     orgId: 1
     folder: ''
     type: file
     disableDeletion: false
     editable: true
     options:
       path: /var/lib/grafana/dashboards/default
dashboards:
  default:
    cluster-stats:
      # Ref: https://grafana.com/dashboards/1621
      gnetId: 1621
      revision: 1
      datasource: Prometheus
    prometheus-stats:
      # Ref: https://grafana.com/dashboards/2
      gnetId: 2
      revision: 2
      datasource: Prometheus
    node-exporter:
      # Ref: https://grafana.com/dashboards/1860
      gnetId: 1860
      revision: 13
      datasource: Prometheus
```

注意这里我添加了3个默认的监控面板，你也可以添加你自己的监控面板（最简单的方式是使用ConfigMap）。

然后部署——
```
helm install ——name grafana ——namespace thanos stable/grafana -f grafana-values.yaml
```

然后用port-forward——
```
kubectl -n thanos port-forward svc/grafana 8080:80
```

哇~ 现在你已经完成了部署，获得了一个高可用的监控解决方案，它基于**Prometheus**，对跨集群的Prometheus提供了一个**集中化的全局视图**，并且拥有**长久保留数据的存储能力**。

## 其他选择

这篇文章主要关注在Prometheus和Thanos，但是如果集中化的全局视图，你不需要，你仍旧可以考虑仅仅使用Prometheus和定义一个持久化存储。

另外的选择是部署[cortex](https://github.com/cortexproject/cortex)，它是另一个开源项目，比Thanos要复杂一些，但是提供了不同的解决思路。




