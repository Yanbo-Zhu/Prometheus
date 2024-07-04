
https://www.cnblogs.com/JetpropelledSnake/p/10768095.html


# 1 Kubernetes Deployment

最后，部署 Prometheus，需要注意的是，我们已经在 k8s 之外单独部署了一套，为了统一处理，在这里是打算作为中转的。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus
  namespace: kube-system
  labels:
    app: prometheus
data:
  prometheus.yml: |-
# 1 省略，在这里定义你需要的配置


---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: prometheus
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        args:
          - '-config.file=/prometheus-data/prometheus.yml'
          # 显然，这里没有用 `Stateful Sets`，存储时间不用太长
          - '-storage.local.retention=48h0m0s'
        ports:
        - name: prometheus
          containerPort: 9090
        volumeMounts:
        - name: data-volume
          mountPath: /prometheus-data
      volumes:
      - name: data-volume
        configMap:
          name: prometheus
---

# 简单处理，直接使用 NodePort 暴露服务，你也可以使用 Ingress
apiVersion: v1
kind: Service
metadata:
  name: prometheus
  namespace: kube-system
spec:
  selector:
    app: prometheus
  ports:
  - name: prometheus
    protocol: TCP
    port: 9090
    nodePort: 30090
  type: NodePort

```


# 2 Prometheus Federate

而在我们外部单独的 Prometheus 中，需要配置 [Federate](https://prometheus.io/docs/operating/federation/)，将 k8s 中 Prometheus 采集的 metrics 全部同步出来。

```
- job_name: 'federate'
  scrape_interval: 15s

  honor_labels: true
  metrics_path: '/federate'

  params:
    'match[]':
      - '{job=~".+"}' # 取 k8s 里面部署的  Prometheus 中所有的 job 数据
  static_configs:
    - targets:
      - '<k8s-node1>:30090'
      - '<k8s-node2>:30090'
      - '<k8s-node3>:30090'
```


# 3 Kubernetes SD

其次，上次提到，我们可以用到 Service Discovery 这个功能，其中就包含 [Kubernetes SD](https://prometheus.io/docs/operating/configuration/#%3Ckubernetes_sd_config)。

它包含四种角色：

- node
- service
- pod
- endpoints

由于篇幅所限，这里只是简单介绍下其中的 node 还有 pod 角色：

```
- job_name: 'kubernetes-nodes'
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  
  kubernetes_sd_configs:
  - role: node
  
  relabel_configs:
    # 即从 __meta_kubernetes_node_label_<labelname> 这个配置中取出 labelname 以及 value
  - action: labelmap
    regex: __meta_kubernetes_node_label_(.+)
    
    # 配置 address 为 k8s api 的地址，相关的 ca 证书以及 token 在上面配置
  - target_label: __address__
    replacement: kubernetes.default.svc:443
    
    # 取出所有的 node，然后设置 /api/v1/nodes/<node_name>/proxy/metrics 为 metrics path
  - source_labels: 
    - __meta_kubernetes_node_name
    regex: (.+)
    target_label: __metrics_path__
    replacement: /api/v1/nodes/${1}/proxy/metrics
```

接下来的这个 pod 角色挺重要：

```
- job_name: 'kubernetes-pods'

  kubernetes_sd_configs:
  - role: pod

  relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
    action: replace
    target_label: __metrics_path__
    regex: (.+)
  - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
    action: replace
    regex: ([^:]+)(?::\d+)?;(\d+)
    replacement: $1:$2
    target_label: __address__
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)
  - source_labels: [__meta_kubernetes_namespace]
    action: replace
    target_label: kubernetes_namespace
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: kubernetes_pod_name
```


在定义了这个角色之后，你只要在你部署的应用 Pod 描述中，加入以下 annotations 就能让 Prometheus 自动发现此 Pod 并采集监控数据了：

```
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "<your app port>"
```

