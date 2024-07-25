
在第8章章中，为了在Kubernetes能够方便的管理和部署Prometheus，我们使用ConfigMap了管理Prometheus配置文件。每次对Prometheus配置文件进行升级时，，我们需要手动移除已经运行的Pod实例，从而让Kubernetes可以使用最新的配置文件创建Prometheus。 而如果当应用实例的数量更多时，通过手动的方式部署和升级Prometheus过程繁琐并且效率低下。

从本质上来讲Prometheus属于是典型的有状态应用，而其有包含了一些自身特有的运维管理和配置管理方式。而这些都无法通过Kubernetes原生提供的应用管理概念实现自动化。为了简化这类应用程序的管理复杂度，CoreOS率先引入了Operator的概念，并且首先推出了针对在Kubernetes下运行和管理Etcd的Etcd Operator。并随后推出了Prometheus Operator。

> 简言之，Prometheus Operator能够帮助用户自动化的创建以及管理Prometheus Server以及其相应的配置。

==The Prometheus Operator acts as a Controller for the Custom Resources. When you deployed the **Prometheus** resource, the Operator created the Prometheus instance, that you just identified when getting a list Pods in the **prometheus** namespace.==

> 通过 Prometheus Operator  能够安装一个 resource ， 这个 resource 的名字为 Prometheus。。像 POD, Depolyment 是resouce 的名字 
Prometheus Operator  就是一个 Controller for the Custom Resources.， 这个 Custom Resources 就是 名字为 Prometheus 的resource 



The Prometheus Operator GitHub project provides a [full set](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md) of API documentation that defines the fields that can be set on the CRDs that it provides. You can see the specification for the **Prometheus** **Kind** [here](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#prometheusspec).


# 1 Prometheus Operator的工作原理

从概念上来讲Operator就是针对管理特定应用程序的，在Kubernetes基本的Resource和Controller的概念上，以扩展Kubernetes api的形式。帮助用户创建，配置和管理复杂的有状态应用程序。从而实现特定应用程序的常见操作以及运维自动化。

==在Kubernetes中我们使用Deployment、DamenSet，StatefulSet来管理应用Workload，使用Service，Ingress来管理应用的访问方式，使用ConfigMap和Secret来管理应用配置。我们在集群中对这些资源的创建，更新，删除的动作都会被转换为事件(Event)，Kubernetes的Controller Manager负责监听这些事件并触发相应的任务来满足用户的期望。这种方式我们成为声明式，用户只需要关心应用程序的最终状态，其它的都通过Kubernetes来帮助我们完成，通过这种方式可以大大简化应用的配置管理复杂度。==

而除了这些原生的Resource资源以外，Kubernetes还允许用户添加自己的自定义资源(Custom Resource)。并且通过实现自定义Controller来实现对Kubernetes的扩展。

如下所示，是Prometheus Operator的架构示意图：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LTqt7j2IAOToEk6hOsn%252F-LTqt9UGxsEOJ93P6X22%252Fprometheus-architecture.png%3Fgeneration%3D1544961698797610%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=685c96e6&sv=1)

Prometheus Operator架构

Prometheus的本职就是一组用户自定义的CRD资源以及Controller的实现，Prometheus Operator负责监听这些自定义资源的变化，并且根据这些资源的定义自动化的完成如Prometheus Server自身以及配置的自动化管理工作。


# 2 Prometheus Operator能做什么

要了解Prometheus Operator能做什么，其实就是要了解Prometheus Operator为我们提供了哪些自定义的Kubernetes资源，列出了Prometheus Operator目前提供的️4类资源：

- Prometheus：声明式创建和管理Prometheus Server实例；
- ServiceMonitor：负责声明式的管理监控配置；
- PrometheusRule：负责声明式的管理告警配置；
- Alertmanager：声明式的创建和管理Alertmanager实例。

> 简言之，Prometheus Operator能够帮助用户自动化的创建以及管理Prometheus Server以及其相应的配置。


# 3 在Kubernetes集群中安装Prometheus Operator


## 3.1 使用 code 

在Kubernetes中安装Prometheus Operator非常简单，用户可以从以下地址中过去Prometheus Operator的源码：
```
git clone https://github.com/coreos/prometheus-operator.git
```


这里，我们为Promethues Operator创建一个单独的命名空间monitoring：
```
kubectl create namespace monitoring
```



由于需要对Prometheus Operator进行RBAC授权，而默认的bundle.yaml中使用了default命名空间，因此，在安装Prometheus Operator之前需要先替换一下bundle.yaml文件中所有namespace定义，由default修改为monitoring。 
通过运行一下命令安装Prometheus Operator的Deployment实例：
```
$ kubectl -n monitoring apply -f bundle.yaml
clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
clusterrole.rbac.authorization.k8s.io/prometheus-operator created
deployment.apps/prometheus-operator created
serviceaccount/prometheus-operator created
service/prometheus-operator created
```

Prometheus Operator通过Deployment的形式进行部署，为了能够让Prometheus Operator能够监听和管理Kubernetes资源同时也创建了单独的ServiceAccount以及相关的授权动作。

查看Prometheus Operator部署状态，以确保已正常运行：
```
$ kubectl -n monitoring get pods
NAME                                   READY     STATUS    RESTARTS   AGE
prometheus-operator-6db8dbb7dd-2hz55   1/1       Running   0          19s
```



## 3.2 使用 Helm-Chart

https://observability.thomasriley.co.uk/prometheus/deploying-prometheus/deploying-prometheus-operator/

![](image/Pasted%20image%2020240712124406.png)


First we will use the community maintained [Helm chart](https://github.com/helm/charts/tree/master/stable/prometheus-operator) for deploying Prometheus Operator to Kubernetes. By default, the Helm chart will also deploy and configure an instance of Prometheus however to begin with lets deploy a standalone instance of the Operator.


1  values.yaml
Lets modify the default behavior of the Helm chart. Create a file called **values.yaml** containing the following:

```
defaultRules:
  create: false
alertmanager:
  enabled: false
grafana:
  enabled: false
kubeApiServer:
  enabled: false
kubelet:
  enabled: false
kubeControllerManager:
  enabled: false
coreDns:
  enabled: false
kubeEtcd:
  enabled: false
kubeScheduler:
  enabled: false
kubeStateMetrics:
  enabled: false
nodeExporter:
  enabled: false
prometheus:
  enabled: false
```



2 install 

Then install the Prometheus Operator via Helm using the **helm upgrade** command as shown below:

```shell
helm upgrade --install prometheus-operator stable/prometheus-operator --namespace prometheus --values values.yaml
```

When this executes, Helm will display all of the resources it has successfully created in Kubernetes:

```shell
$ helm upgrade --install prometheus-operator stable/prometheus-operator --namespace prometheus --values values.yaml

Release "prometheus-operator" does not exist. Installing it now.
NAME:   prometheus-operator
LAST DEPLOYED: Tue Jun 25 22:06:52 2019
NAMESPACE: prometheus
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRole
NAME                              AGE
prometheus-operator-operator      1s
prometheus-operator-operator-psp  1s

==> v1/ClusterRoleBinding
NAME                              AGE
prometheus-operator-operator      1s
prometheus-operator-operator-psp  1s

==> v1/Deployment
NAME                          READY  UP-TO-DATE  AVAILABLE  AGE
prometheus-operator-operator  0/1    1           0          1s

==> v1/Pod(related)
NAME                                           READY  STATUS             RESTARTS  AGE
prometheus-operator-operator-694f88774b-q4r64  0/1    ContainerCreating  0         1s

==> v1/Service
NAME                          TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)   AGE
prometheus-operator-operator  ClusterIP  10.11.250.245  <none>       8080/TCP  1s

==> v1/ServiceAccount
NAME                          SECRETS  AGE
prometheus-operator-operator  1        1s

==> v1/ServiceMonitor
NAME                          AGE
prometheus-operator-operator  1s

==> v1beta1/PodSecurityPolicy
NAME                          PRIV   CAPS      SELINUX   RUNASUSER  FSGROUP    SUPGROUP  READONLYROOTFS  VOLUMES
prometheus-operator-operator  false  RunAsAny  RunAsAny  MustRunAs  MustRunAs  false     configMap,emptyDir,projected,secret,downwardAPI,persistentVolumeClaim


NOTES:
The Prometheus Operator has been installed. Check its status by running:
  kubectl --namespace prometheus get pods -l "release=prometheus-operator"

Visit https://github.com/coreos/prometheus-operator for instructions on how
to create & configure Alertmanager and Prometheus instances using the Operator.

```

Above you can see that Helm has deployed the **stable/prometheus-operator** Helm chart under the release name **prometheus-operator** into the Kubernetes namespace **prometheus** using the Helm values we created above in values.yaml.


3 检查安装成果
If you then use Kubectl to list the Pods in the **prometheus** namespace you will see the Prometheus Operator is now installed:

```shell
$ kubectl get pods -n prometheus
NAME                                            READY   STATUS    RESTARTS   AGE
prometheus-operator-operator-694f88774b-q4r64   1/1     Running   0          6m47s
```


# 4 Launch Prometheus Instance 

When we deployed the Operator, one of the tasks it performed when it first launched was to is install a number Custom Resource Definitions (CRDs) into Kubernetes. Out of the box, Kubernetes ships with many powerful Controllers such as the Deployment or Statefulset. ==CRDs provide a method of building completely bespoke Controllers that provide logic to a specific function.==


## 4.1 安装PrometheusOperator后得到的CRD

In this case, the CRDs installed by Prometheus Operator provide a means for launching and configuring Prometheus within Kubernetes.


If you run the `kubectl get customresourcedefinitions` command in your terminal you will see four CRDs provided by the Operator:

```shell
$kubectl get customresourcedefinitions
NAME                                           CREATED AT
alertmanagers.monitoring.coreos.com            2019-07-02T13:13:21Z
prometheuses.monitoring.coreos.com             2019-07-02T13:13:21Z
prometheusrules.monitoring.coreos.com          2019-07-02T13:13:21Z
servicemonitors.monitoring.coreos.com          2019-07-02T13:13:21Z
```


## 4.2 使用其中一个 CRD去安装一个prometheusInstance 

To begin with we will be making use of the prometheuses.monitoring.coreos.com Custom Resource Definition.

Create a file called **prometheus.yaml** and add the following:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: prometheus
spec:
  baseImage: quay.io/prometheus/prometheus
  logLevel: info
  podMetadata:
    annotations:
      cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
    labels:
      app: prometheus
  replicas: 1
  resources:
    limits:
      cpu: 1
      memory: 2Gi
    requests:
      cpu: 1
      memory: 2Gi
  retention: 12h
  serviceAccountName: prometheus-service-account
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

Now apply this YAML to your Kubernetes cluster using `kubectl apply -f prometheus.yaml`. Kubectl will show that it has successfully created the configuration, as shown below:

```shell
$kubectl apply -f prometheus.yaml
prometheus.monitoring.coreos.com/prometheus created
serviceaccount/prometheus-service-account created
clusterrole.rbac.authorization.k8s.io/prometheus-cluster-role created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-cluster-role-binding created
```


## 4.3 查看安装结果 

Success! If you now list Pods in the **prometheus** namespace using `kubectl get pods --namespace prometheus` you will see an instance of Prometheus running alongside the Prometheus Operator:

```shell
$kubectl get pods
NAME                                            READY   STATUS    RESTARTS   AGE
prometheus-operator-operator-86bc4d5568-shs69   1/1     Running   0          10m
prometheus-prometheus-0                         3/3     Running   0          1m
```



If you now check for the custom Prometheus resource that you have just installed into the cluster using `kubectl get prometheus --namespace prometheus` you will see the single result named **prometheus**:

```shell
$kubectl get prometheus --namespace prometheus
NAME         AGE
prometheus   6m
```

==The Prometheus Operator acts as a Controller for the Custom Resources. When you deployed the **Prometheus** resource, the Operator created the Prometheus instance, that you just identified when getting a list Pods in the **prometheus** namespace.==




# 5 详解prometheus.yaml




```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: prometheus
spec:
  baseImage: quay.io/prometheus/prometheus
  logLevel: info
  podMetadata:
    annotations:
      cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
    labels:
      app: prometheus
  replicas: 1
  resources:
    limits:
      cpu: 1
      memory: 2Gi
    requests:
      cpu: 1
      memory: 2Gi
  retention: 12h
  serviceAccountName: prometheus-service-account
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


## 5.1 概况

Lets now take a look at the **prometheus.yaml** file we applied to Kubernetes and see what each section means.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: prometheus
```

Here we define that we wish to create an object called **prometheus** that is of the type **Prometheus** as defined by the Kind and that this Kind is part of the API Version **monitoring.coreos.com/v1**, that was previously installed into Kubernetes by the Prometheus Operator as a Custom Resource Definition. This object will be created in the **prometheus** namespace.



## 5.2 Spec 


Everything then under the **spec** of the YAML file defines what the instance of Prometheus should look like.


1 
```yaml
  replicas: 1
  resources:
    limits:
      cpu: 1
      memory: 2Gi
    requests:
      cpu: 1
      memory: 2Gi
```

Here we define the Resource limits (CPU & Memory) that each Prometheus Pod will be granted within Kubernetes. We also specify the number of instances that we require by setting **replicas**, in this example we have just the 1 instance.


---
2 

```yaml
  baseImage: quay.io/prometheus/prometheus
  version: v2.10.0
```

Setting the **baseImage** defines the actual Prometheus Docker image to be used. This will actually be defaulted to the Docker image that is released by the Prometheus project however we included it as an example. The **version** field sets the version of Prometheus you wish to use. You can see available versions on the [GitHub project](https://github.com/prometheus/prometheus/releases).

---

3  storage

```yaml
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
```

This block defines the storage that will be used by Prometheus. By default the Operator will create Prometheus Pods that use local storage only by using an [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir). If you wish to retain the state of Prometheus and therefore the metrics that it stores when re-launching Prometheus, such as during a version upgrade, then you need to use persistent storage. 

The **PersistentVolumeClaim (PVC)** defines the specification of the storage to be used by Prometheus. In this example we are creating a persistent disk that is 10Gi for each instance of Prometheus that is created.

In your terminal if you execute `kubectl get persistentvolumeclaim --namespace prometheus` you will see the PVC that has been created and Bound to a Persistent Volume for the single instance of Prometheus that you have created:

```shell
$kubectl get persistentvolumeclaim --namespace prometheus
NAME                                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
prometheus-pvc-prometheus-prometheus-0   Bound    pvc-c85b2a3b-9d7a-11e9-9e3c-42010a840079   10Gi       RWO            standard       21m
```



---
4 

The **ServiceAccount**, **ClusterRoleBinding** and **ClusterRoleBinding** are required for providing the Prometheus Pod with the required permissions to access the Kubernetes API as part of its service discovery process.

```yaml
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


---

5 

Lastly lets look at some of the Prometheus specific configuration of the **prometheus.yaml** file:

```yaml
  logLevel: info
  retention: 12h
```

Here we define that Prometheus should retain 12 hours of metrics and that it should log using the **info** log level.















