https://observability.thomasriley.co.uk/prometheus/using-thanos/


The [Thanos Project](https://github.com/improbable-eng/thanos) turns Prometheus into a highly available metrics platform with unlimited metrics storage.

The three key features of Thanos, are as follows:

- Global query view of all metrics from as many Prometheus instances as you require.
- Long term storage of metrics.
- High availability of Prometheus.


# 1 High Availability


Out of the box Prometheus does not have any concept of high availability or redundancy. Prometheus itself may be a mature and reliable product but not everything is foolproof and you should always plan for _when_ a Kubernetes worker node fails, not _if_ it fails, and therefore we must be able to tolerate a Prometheus Pod restarting from time to time. Before we look at Thanos, lets see how we could tackle this problem with just Kubernetes & Prometheus.

## 1.1 High Availability with Kubernetes

Earlier in this chapter we used the Prometheus Operator to launch a single instance of Prometheus within Kubernetes. To avoid the scenario of metrics being unavailable, either permanently or for a short duration of time, we can run a second instance of Prometheus. Each instance of Prometheus will run independent of the other, however each still have the same configuration as set by the Prometheus Operator. Essentially, two copies of target metrics will be scraped, as shown below:

[![Two Prometheus Instances](https://observability.thomasriley.co.uk/prometheus/using-thanos/high-availability/images/multiple-prometheus.png?classes=shadow&width=30pc)](https://observability.thomasriley.co.uk/prometheus/using-thanos/high-availability/images/multiple-prometheus.png?classes=shadow&width=30pc)


### 1.1.1 部署两个 Prometheus Instance

Now lets update the previous **prometheus.yaml** file to support this new architecture. We need to change the **replicas** from 1 to 2 and then also add a **podAntiAffinity** to ensure that both instances of Prometheus are running on different Kubernetes worker nodes to ensure we truly benefit from having the additional redundant instance of Prometheus.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: prometheus
spec:
  affinity:  # 注意这里
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - prometheus
          topologyKey: kubernetes.io/hostname
  baseImage: quay.io/prometheus/prometheus
  logLevel: info
  podMetadata:
    annotations:
      cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
    labels:
      app: prometheus
  replicas: 2   # 注意这里
  resources:
    limits:
      cpu: 1
      memory: 2Gi
    requests:
      cpu: 1
      memory: 2Gi
  retention: 12h
  serviceAccountName: prometheus-service-account
  serviceMonitorSelector:
    matchLabels:
      serviceMonitorSelector: prometheus
  storage:
    volumeClaimTemplate:
      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: prometheus-pvc
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
  version: v2.10.0
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: "prometheus-service-account"
  namespace: "prometheus"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: "prometheus-cluster-role"
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - "/metrics"
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: "prometheus-cluster-role-binding"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: "prometheus-cluster-role"
subjects:
- kind: ServiceAccount
  name: "prometheus-service-account"
  namespace: prometheus
```

Lets apply this updated **prometheus.yaml** to Kubernetes by running `kubectl apply -f prometheus.yaml`.

A moment or two after applying this, check the running Pods in the **prometheus** namespace by running `kubectl get pods --namespace prometheus`:

```shell
$kubectl get pods --namespace prometheus
NAME                                            READY   STATUS    RESTARTS   AGE
prometheus-operator-operator-86bc4d5568-7k6tp   1/1     Running   0          23h
prometheus-prometheus-0                         3/3     Running   0          2d
prometheus-prometheus-1                         3/3     Running   0          1m10s
```


## 1.2 做测试


用port-forward 的时候使用的是 Kubernetes internal load balancing functionality,, 去连接 2个 prometheus instance 中其中的一个， 但你不知道连接的是其中的哪一个 

Now lets put the reliability of this to the test:

- Reconnect to Prometheus by executing `kubectl port-forward service/prometheus-operated 9090:9090 --namespace prometheus` and then access the UI at [https://localhost:9090](http://localhost:9090)
- Restart one of the instances of Prometheus by running `kubectl delete pod prometheus-prometheus-0 --namespace prometheus`
- Immediately then check the Prometheus UI in your web browser, you will see that it is still available!


This is great however there is one thing we need to think about. If we have two instances of Prometheus with two copies of of the same metrics, which should we use?


Your Prometheus deployment uses a Kubernetes Service and in the previous example you used Kubectl port-forwarding, connecting to the Kubernetes Service directly, and therefore taking advantage of Kubernetes internal load balancing functionality. You have essentially have implemented the illustration below:

[![Two Prometheus Instances with Kubernetes Service](https://observability.thomasriley.co.uk/prometheus/using-thanos/high-availability/images/multiple-prometheus-with-service.png?classes=shadow&width=30pc)](https://observability.thomasriley.co.uk/prometheus/using-thanos/high-availability/images/multiple-prometheus-with-service.png?classes=shadow&width=30pc)

用port-forward 去连接 2个 prometheus instance 中其中的一个， 但你不知道连接的是其中的哪一个 ， 也无法指定某个instnce 
==So when you connect to Prometheus via the Kuberneres Service the request will be serviced by one of the running Prometheus instances. However, when you make subsequent requests there is no guarantee that the request will be serviced by the same instance. ==


上图这种模式下 ， 无法保障 两个 instance scrape 重要的 metrice in the same time, 他们会抢夺 metrice , 之间不同步 ， 所以 你用 grafana 去看数据的时候， 同一个 graph, 设置的查询的时间段相同， 也会看到不同数据。 
上面这缺点时可用 Thanos 来消除的 
Why is this an issue? The two instances of Prometheus that are running are independent of each other and while they do have the same scrape configuration there is no guarantee that they will scrape the targets at exactly the same time, therefore the time series metrics that they each collect may have different values.

What this all means is, each time you connect to Prometheus via the load balanced Kubernetes Service, you may see some oddness with metrics changing. When visualizing the metrics over time with dashboarding tools such as Grafana, this leads to a really poor experience for users, as each time you reload the same graph it may appear differently in the same time period. This is now where Thanos can help!


## 1.3 High Availability with Thanos

At a high level, HA for Prometheus with Thanos works as detailed below:

- First a sidecar is deployed alongside the Prometheus container and interacts with Prometheus. ==A sidecar is an additional container within the Kubernetes Pod running alongside other containers.==
- Next, an additional service is deployed called Thanos Query and is configured to be aware of of all instances of the Thanos Sidecar.
- Thanos Query communicates with the Thanos Sidecar via [gRPC](https://grpc.io/) and de-duplicates metrics across all instances of Prometheus when executing a query. Query exposes users to a Prometheus-esuqe user interfance and also exposes the Prometheus API.

The diagram below shows this:

[![Two Prometheus Instances with Thanos Sidecar](https://observability.thomasriley.co.uk/prometheus/using-thanos/high-availability/images/multiple-prometheus-with-thanos.png?classes=shadow&width=30pc)](https://observability.thomasriley.co.uk/prometheus/using-thanos/high-availability/images/multiple-prometheus-with-thanos.png?classes=shadow&width=30pc)

### 1.3.1 如何implment 上面的模型: 修改**prometheus.yaml**

1  加入 internal label 
The Prometheus Operator supports the configuration of the Thanos Sidecar via the Prometheus CRD, so you simply need to update the existing deployment. You then also need to deploy the Thanos Query service and configure this to federate the instances of Prometheus that are running. 
You will use a Kubernetes Service as a mechanism of service discovery for configuring Thanos Query to identify the Prometheus instances, to do this ==you also need to add an additional Kubernetes Pod Label to the Prometheus Pods so you can select them with the label selector on the Kubernetes Service.==


2 加入 external label
Finally, you must also set an external label for the Prometheus instances to use. The external label is required by Thanos and is used as a method of labelling all metrics that are derived from a particular instance of Thanos Query.

3  spec 怎么写： 在**prometheus.yaml** 中kind: Prometheus对应的spec 加入下面的内容
Update the Prometheus resource adding the **thanos** configuration, the new service discovery label, and also configure an external label for the Prometheus instances:


```yaml
spec:
  podMetadata:
    labels: 
      thanos-store-api: "true"  # 注意这里 
  thanos:
    version: v0.4.0
    resources:
      limits:
        cpu: 500m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 500Mi
  externalLabels:
    cluster_environment: workshop  # 注意这里 
```


4 在 **prometheus.yaml**  最下面加入下面的内容 
define a Kubernetes Deployment for Thanos Query
define the Kubernetes Service for the purposes of service discovery 

Then define a Kubernetes Deployment for Thanos Query and the Kubernetes Service for the purposes of service discovery by adding the below to **prometheus.yaml** also:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-query
  namespace: prometheus
  labels:
    app: thanos-query
spec:
  replicas: 1
  selector:
    matchLabels:
      app: thanos-query
  template:
    metadata:
      labels:
        app: thanos-query
    spec:
      containers:
      - name: thanos-query
        image: improbable/thanos:v0.5.0
        resources:
          limits:
            cpu: 500m
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 500Mi
        args:
        - "query"
        - "--log.level=debug"
        - "--query.replica-label=prometheus_replica"
        - "--store.sd-dns-resolver=miekgdns"
        - "--store=dnssrv+_grpc._tcp.thanos-store-api.prometheus.svc.cluster.local"
        ports:
        - name: http
          containerPort: 10902
        - name: grpc
          containerPort: 10901
        - name: cluster
          containerPort: 10900
---
apiVersion: v1
kind: Service
metadata:
  name: "thanos-store-api"
  namespace: prometheus
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: grpc
    port: 10901
    targetPort: grpc
  selector:
    thanos-store-api: "true"
```

The updated **prometheus.yaml** file should be similar to the example [here](https://observability.thomasriley.co.uk/prometheus/using-thanos/high-availability/static/prometheus-with-sidecar.yaml).


### 1.3.2 depolyed 修改后的**prometheus.yaml**



Now apply this to Kubernetes by executing `kubectl apply -f prometheus.yaml`. You wil see two things occur, the new Thanos Query service will be deployed as a Pod and the existing Prometheus Pods will be restarted since the new Pod Label and Sidecar are to be added:

```shell
$kubectl get pods --namespace prometheus
NAME                                            READY   STATUS              RESTARTS   AGE
prometheus-operator-operator-86bc4d5568-94cpf   1/1     Running             0          18m
prometheus-prometheus-0                         0/4     ContainerCreating   0          1s
prometheus-prometheus-1                         0/4     Pending             0          1s
thanos-query-58bcc6dcbb-67rn4                   0/1     ContainerCreating   0          4s

$kubectl get pods --namespace prometheus
NAME                                            READY   STATUS              RESTARTS   AGE
prometheus-operator-operator-86bc4d5568-94cpf   1/1     Running             0          19m
prometheus-prometheus-0                         4/4     Running             0          1m
prometheus-prometheus-1                         4/4     Running             0          1m
thanos-query-58bcc6dcbb-67rn4                   1/1     Running             0          1m
```

You can see that Pods **prometheus-prometheus-0** and **prometheus-prometheus-1** now show **4/4** on the container readiness. Previously this only showed 3 containers, but following this change there are now 4 containers in a Prometheus Pod due to the additional Thanos Sidecar.


### 1.3.3 连接Thanos Query UI


Now connect to Thanos Query using port forwarding by executing a Kubectl command. You will need to substitute the correct name of your Thanos Query Pod. For example: `kubectl port-forward pod/thanos-query-58bcc6dcbb-67rn4 10902:10902 --namespace prometheus`.


When accessing [http://localhost:10902](http://localhost:10902) in your web browser you will see that the Thanos Query UI is awfully similar to the Prometheus UI. That is no accident, Thanos is actually based on the same codebase as Prometheus.

[![Thanos Query](https://observability.thomasriley.co.uk/prometheus/using-thanos/high-availability/images/thanos-graph.png?classes=shadow&width=30pc)](https://observability.thomasriley.co.uk/prometheus/using-thanos/high-availability/images/thanos-graph.png?classes=shadow&width=30pc)

When running a Prometheus query in Thanos you can see a checkbox named **deduplication**. If you experiment running Prometheus queries with this option enabled and disable you will see how Thanos deduplicates the metrics in the available Prometheus instances when querying.

If you select the **Stores** option in the menu at the top, Thanos Query has a interface for showing the Thanos Store API endpoints it is currently federating. When you check this, you will see the two Prometheis intances that are running, as shown below:

[![Thanos Stores](https://observability.thomasriley.co.uk/prometheus/using-thanos/high-availability/images/thanos-stores.png?classes=shadow&width=30pc)](https://observability.thomasriley.co.uk/prometheus/using-thanos/high-availability/images/thanos-stores.png?classes=shadow&width=30pc)




# 2 Long Term Storage

Now that you have enabled high availability with Prometheus using Thanos you can look at the next killer feature of Thanos, long term storage of metrics!

To enable this, 
1 you first need to create a bucket on an object store such as AWS S3 or GCP Storage. 
2 Next you enable the Thanos Sidecar to upload metrics to the object store.
3 Lastly, you need to deploy another Thanos component called Store that acts as an API for the metrics available in the object store and will be queried by the existing Thanos Query instance when a user executes a Prometheus query via Thanos Query.

The diagram below shows this using a GCP Storage bucket.


![](image/long-term-storage.png)


From the perspective of users they are completely unaware that when they execute a Prometheus query Thanos is querying the for metrics from both the Prometheus instances and from the object storage.



储存在 Prometheus instances  block memory 如何通过 thanos sidecar 把 data 床送个 external object sotrage 

How does this work? Prometheus stores metrics in **blocks**. Initially, it stores a block in memory however periodically, typically every 2 hours, it writes the current in memory block out to the filesystem. As the Thanos Sidecar in the Prometheus Pod has the same shared filesystem as the Prometheus container it can see the new block that Prometheus writes to the filesystem. Once it sees a new block on disk, the Thanos Sidecar uploads the block to the object storage as per its configuration. This is shown below in the diagram.

[![Thanos Sidecar Upload](https://observability.thomasriley.co.uk/prometheus/using-thanos/long-term-storage/images/thanos-sidecar-upload.png?classes=shadow&width=40pc)](https://observability.thomasriley.co.uk/prometheus/using-thanos/long-term-storage/images/thanos-sidecar-upload.png?classes=shadow&width=40pc)


## 2.1 implementation 

前期准备
Lets now implement this. For this example 
1 I created a Google Cloud Storage bucket called **observability-for-kubernetes-thanos-demo** and 
2 provisioned a Service Account with ‘Storage Admin’ permissions so that Thanos can read and write to the storage bucket. 
3 In the cloud provider of your choice, create a storage bucket and set of credentials with permission to access the bucket.
## 2.2 Kubernetes Secret that contains the GCP Service Account and Bucket Name. 

We you need to create a Thanos Object Store configuration file that provides the bucket configuration and credentials to Thanos. The structure of this file differs per cloud provider, you can see the [Thanos Documenation](https://github.com/improbable-eng/thanos/blob/master/docs/storage.md) to see the different options but for this example lets proceed using Google Cloud Platform.


然后根据 这个 Thanos Object Store configuration file You will create a Kubernetes Secret that contains the GCP Service Account and Bucket Name. 
The example below shows the expected structure, however the GCP Service Account is not valid for obvious security reasons!

Thanos Object Store configuration file: 
```
type: GCS
config:
  bucket: "observability-for-kubernetes-thanos-demo"
  service_account: |-
  {
    "type": "service_account",
    "project_id": "kubernetes-cloud-lab",
    "private_key_id": "",
    "private_key": "-----BEGIN PRIVATE KEY-----\n\n-----END PRIVATE KEY-----\n",
    "client_email": "observability-for-kubernetes@kubernetes-cloud-lab.iam.gserviceaccount.com",
    "client_id": "",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/observability-for-kubernetes%40kubernetes-cloud-lab.iam.gserviceaccount.com"
  }  

```

---


用thanos-object-secret.yaml 去产生  Kubernetes Secret that contains the GCP Service Account and Bucket Name： 
You should take the valid version of the example above and base64 encode it. You can use base64 in your terminal for doing this. Then create a file called thanos-object-secret.yaml and include the Kubernetes Secret, as shown below:

```
---
apiVersion: v1
kind: Secret
metadata:
  name: thanos-config
  namespace: prometheus
data:
  thanos.config: dHlwZTogR0NTCmNvbmZpZzoKICBidWNrZXQ6ICJvYnNlcnZhYmlsaXR5LWZvci1rdWJlcm5ldGVzLXRoYW5vcy1kZW1vIgogIHNlcnZpY2VfYWNjb3VudDogfC0KICB7CiAgICAidHlwZSI6ICJzZXJ2aWNlX2FjY291bnQiLAogICAgInByb2plY3RfaWQiOiAia3ViZXJuZXRlcy1jbG91ZC1sYWIiLAogICAgInByaXZhdGVfa2V5X2lkIjogIiIsCiAgICAicHJpdmF0ZV9rZXkiOiAiLS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tXG5cbi0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS1cbiIsCiAgICAiY2xpZW50X2VtYWlsIjogIm9ic2VydmFiaWxpdHktZm9yLWt1YmVybmV0ZXNAa3ViZXJuZXRlcy1jbG91ZC1sYWIuaWFtLmdzZXJ2aWNlYWNjb3VudC5jb20iLAogICAgImNsaWVudF9pZCI6ICIiLAogICAgImF1dGhfdXJpIjogImh0dHBzOi8vYWNjb3VudHMuZ29vZ2xlLmNvbS9vL29hdXRoMi9hdXRoIiwKICAgICJ0b2tlbl91cmkiOiAiaHR0cHM6Ly9vYXV0aDIuZ29vZ2xlYXBpcy5jb20vdG9rZW4iLAogICAgImF1dGhfcHJvdmlkZXJfeDUwOV9jZXJ0X3VybCI6ICJodHRwczovL3d3dy5nb29nbGVhcGlzLmNvbS9vYXV0aDIvdjEvY2VydHMiLAogICAgImNsaWVudF94NTA5X2NlcnRfdXJsIjogImh0dHBzOi8vd3d3Lmdvb2dsZWFwaXMuY29tL3JvYm90L3YxL21ldGFkYXRhL3g1MDkvb2JzZXJ2YWJpbGl0eS1mb3Ita3ViZXJuZXRlcyU0MGt1YmVybmV0ZXMtY2xvdWQtbGFiLmlhbS5nc2VydmljZWFjY291bnQuY29tIgogIH0=

```
The value of thanos.config should be set to the base64 string that was just generated.

---

Apply thanos-object-secret.yaml to Kubernetes by executing kubectl apply -f thanos-object-secret.yaml:

```
$kubectl apply -f thanos-object-secret.yaml
secret/cluster-thanos-config created

```


## 2.3 update the Prometheus resource adding the **objectStorageConfig** key into **prometheus.yaml**

With the object store configuration deployed to Kubernetes, next update the Prometheus resource adding the **objectStorageConfig** key so that the Prometheus Operator configures the Thanos Sidecar with the object storage config that has just been deployed to Kubernetes.

The Prometheus resource should look similar to the below with this added:

```
---
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: prometheus
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - prometheus
          topologyKey: kubernetes.io/hostname
  baseImage: quay.io/prometheus/prometheus
  logLevel: info
  podMetadata:
    annotations:
      cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
    labels:
      app: prometheus
      thanos-store-api: "true"
  replicas: 2
  thanos:
    version: v0.4.0
    resources:
      limits:
        cpu: 500m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 500Mi
    objectStorageConfig:   # 注意这里
      key: thanos.config
      name: thanos-config
  resources:
    limits:
      cpu: 1
      memory: 2Gi
    requests:
      cpu: 1
      memory: 2Gi
  retention: 12h
  serviceAccountName: prometheus-service-account
  serviceMonitorSelector:
    matchLabels:
      serviceMonitorSelector: prometheus
  externalLabels:
    cluster_environment: workshop
  storage:
    volumeClaimTemplate:
      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: prometheus-pvc
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
  version: v2.10.0

```


Edit the **prometheus.yaml** file you created previously to reflect the changes above. Under the **objectStorageConfig** key set the **name** to be the name of the Kubernetes Secret you just created and the **key** to be the name of the secret key you used, which in this case is **thanos.config**.

Apply the updated **prometheus.yaml** to Kubernetes by running `kubectl apply -f prometheus.yaml`. To see an example of the full YAML file that reflects the changes described above see [here](https://observability.thomasriley.co.uk/prometheus/using-thanos/long-term-storage/static/thanos-with-object-config.yaml).

## 2.4 验收结果 

Once the Prometheus Pods have restarted with the new configuration, use `kubectl logs` to view the logs from the **thanos-sidecar** container in one of the Prometheus Pods, as shown below:

```shell
$kubectl logs prometheus-prometheus-0 thanos-sidecar --namespace prometheus
level=info ts=2019-07-05T20:48:04.403642465Z caller=flags.go:87 msg="gossip is disabled"
level=info ts=2019-07-05T20:48:04.403893193Z caller=main.go:257 component=sidecar msg="disabled TLS, key and cert must be set to enable"
level=info ts=2019-07-05T20:48:04.403923795Z caller=factory.go:39 msg="loading bucket configuration"
level=info ts=2019-07-05T20:48:04.404486705Z caller=sidecar.go:319 msg="starting sidecar" peer="no gossip"
level=info ts=2019-07-05T20:48:04.404636222Z caller=main.go:309 msg="Listening for metrics" address=0.0.0.0:10902
level=info ts=2019-07-05T20:48:04.404674065Z caller=reloader.go:154 component=reloader msg="started watching config file and non-recursively rule dirs for changes" cfg= out= dirs=
level=info ts=2019-07-05T20:48:04.40474036Z caller=sidecar.go:260 component=sidecar msg="Listening for StoreAPI gRPC" address=[10.8.2.14]:10901
level=info ts=2019-07-05T20:48:06.414276291Z caller=sidecar.go:176 msg="successfully loaded prometheus external labels" external_labels="{cluster_environment=\"workshop\",prometheus=\"prometheus/prometheus\",prometheus_replica=\"prometheus-prometheus-0\"}"
level=info ts=2019-07-05T20:48:34.441923855Z caller=shipper.go:350 msg="upload new block" id=01DF1NSPGCHYM8T82R8BKAX4HP
level=info ts=2019-07-05T20:48:35.12152462Z caller=shipper.go:350 msg="upload new block" id=01DF1R3VHXY2NA2ZH4W4TVM33H
level=info ts=2019-07-05T21:00:04.457140171Z caller=shipper.go:350 msg="upload new block" id=01DF1YZJT0GNZ2G0YWJWJ3NE25
```

If your instances of Prometheus have been running long enough (atleast 2 hours) there should be a block on the filesystem ready to be uploaded. If there is, you will see it upload the block to the storage bucket as shown above. See the log entry **“upload new block”** where it begins the upload. If you check the bucket in the cloud provider console you will the blocks are now present.



## 2.5 Thanos Components

Now that the Thanos Sidecar is uploading blocks to the object store, we need to deploy two addtional components: Thanos Store and Thanos Compact.


![](image/long-term-storage.png)

### 2.5.1 Thanos Store

Thanos Store acts as an API for querying Prometheus metrics stored in the object store.

Update **prometheus.yaml** to also include the following:

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: thanos-store
  namespace: prometheus
  labels:
    app: thanos-store
    thanos-store-api: "true"
spec:
  replicas: 1
  serviceName: thanos-store
  selector:
    matchLabels:
      app: thanos-store
      thanos-store-api: "true"
  template:
    metadata:
      labels:
        app: thanos-store
        thanos-store-api: "true"
    spec:
      containers:
      - name: thanos-store  # 注意这里 
        image: improbable/thanos:v0.5.0
        resources:
          limits:
            cpu: 1
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 1Gi
        args:
        - "store"
        - "--data-dir=/prometheus/cache"
        - "--objstore.config-file=/config/thanos.config"
        - "--log.level=info"
        - "--index-cache-size=256MB"
        - "--chunk-pool-size=256MB"
        - "--store.grpc.series-max-concurrency=30"
        ports:
        - name: http
          containerPort: 10902
        - name: grpc
          containerPort: 10901
        - name: cluster
          containerPort: 10900
        volumeMounts:
        - mountPath: /prometheus
          name:  thanos-store-storage
        - mountPath: /config/
          name: thanos-config
      volumes:
      - name: thanos-config
        secret:
          secretName: thanos-config
  volumeClaimTemplates:
  - metadata:
      name: thanos-store-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

Apply this to Kubernetes by running `kubectl apply -f prometheus.yaml`.

Kubernetes will launch a single Thanos Store Pod as per the configuration above. If you check the available **Stores** in the Thanos Query UI, you will now see Thanos Store listed in addition to the two Thanos Sidecars running alongside Prometheus. Now when querying via Thanos Query, the query will execute across both the Prometheus instances and also the metrics stored in the object store!

[![Thanos Query Stores](https://observability.thomasriley.co.uk/prometheus/using-thanos/long-term-storage/images/thanos-query-with-store.png?classes=shadow&width=40pc)](https://observability.thomasriley.co.uk/prometheus/using-thanos/long-term-storage/images/thanos-query-with-store.png?classes=shadow&width=40pc)

### 2.5.2 Thanos Compact

Thanos Compact is the final Thanos component you need to deploy.

Compact performs three main tasks:

- It executes a Prometheus compaction job on the blocks in the object store. Typically Prometheus would execute this for blocks locally on the filesystem but the process is disabled by Prometheus Operator when using Thanos.
- It also executes a down-sampling process on metrics in the object store. Prometheus stores metrics with a resolution of 1 minute. However, if you were to execute a query over a period of months or years on a Prometheus environment using Thanos with long term storage, the number of data-points returned would be excessive! Therefore, Compact performs the down-sampling job adding a 5 minute and 1 hour sample in addition to the 1 minute sample. It does this by creating a new block and discarding the original once down-sampling is completed. When executing queries, Thanos will automatically select the most appropriate sample to return.
- Lastly, it is also possible to set retention periods for the 1 minute, 5 minute and 1 hour samples. Compact will apply these retentions if they are set.

Now lets deploy Thanos Compact.

Update **prometheus.yaml** to also include the following:

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: thanos-compact
  namespace: prometheus
  labels:
    app: thanos-compact
spec:
  replicas: 1
  serviceName: thanos-compact
  selector:
    matchLabels:
      app: thanos-compact
  template:
    metadata:
      labels:
        app: thanos-compact
    spec:
      containers:  # 注意这里
      - name: thanos-compact
        image: improbable/thanos:v0.5.0
        resources:
          limits:
            cpu: 1
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 1Gi
        args:
        - "compact"
        - "--data-dir=/prometheus/compact"
        - "--objstore.config-file=/config/thanos.config"
        - "--log.level=info"
        - "--retention.resolution-raw=2d"
        - "--retention.resolution-5m=5d"
        - "--retention.resolution-1h=10d"
        - "--consistency-delay=15m"
        - "--wait"
        ports:
        - name: http
          containerPort: 10902
        - name: grpc
          containerPort: 10901
        - name: cluster
          containerPort: 10900
        volumeMounts:
        - mountPath: /prometheus
          name: thanos-compact-storage
        - mountPath: /config/
          name: thanos-config
      volumes:
      - name: thanos-config
        secret:
          secretName: thanos-config
  volumeClaimTemplates:
  - metadata:
      name: thanos-compact-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

Apply this to Kubernetes by running `kubectl apply -f prometheus.yaml`.

Kubernetes will launch a single Thanos Compact Pod which will, once running, start performing the actions described above on blocks stored in the object store.

To see an example of the full YAML file that reflects the changes described above with Thanos Store and Compact see [here](https://observability.thomasriley.co.uk/prometheus/using-thanos/long-term-storage/static/thanos-with-components.yaml).















