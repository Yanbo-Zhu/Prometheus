
metrice 的label的来源
- application 自己在产生标签的时候， 就会自动给自己配上一些特定的label 和这个label的value
    - 比如说 `application_core_cluster_payloads_received_latencies_ms{server="d3227a-ivuplan-foreground-575d769947-tfdjd",quantile="0.5"} 0.0`
- prometheus 收集metrics的时候， 会贴上 新的label 

# 1 set a label in Service monitor so it appears in Prometheus metrics

https://stackoverflow.com/questions/63852779/how-to-set-a-label-in-service-monitor-so-it-appears-in-prometheus-metrics

通过 targetLabels in ServiceMonitor使得某些标签被贴在了  通过prometheus 收集到的metrics 上

```yaml
kind: Service
apiVersion: v1
metadata:
  name: example-application
  labels:
    app: example-application
    teamname: neon
spec:
  selector:
    app: example-application
  ports:
  - name: backend
    port: 8080
```


```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-application
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: example-application
  endpoints:
  - port: backend
    path: /prometheus
  namespaceSelector:
    matchNames:
    - testns
  targetLabels:
    - teamname
    - app
```


