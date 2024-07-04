
监控告警实现需要依赖 Alertmanager，已经相关的组件，比如上述实例中的监控服务器应用的node_server组件。

# 1 例子

## 1.1 邮件告警实现

需要安装Alertmanager，这里因为邮件发送比较简单，所以这里我就直接贴配置了，其中带有xxx字符的参数是需要根据情况进行更改的。下面的企业微信告警同理。  
Alertmanagers服务的**alertmanager.yml**的配置如下:

```yaml
global:
  resolve_timeout: 5m
  smtp_from: 'xxx@qq.com'
  smtp_smarthost: 'smtp.qq.com:465'
  smtp_auth_username: 'xxx@qq.com'
  smtp_auth_password: 'xxx'
  smtp_require_tls: false
  smtp_hello: 'qq.com'
route:
  group_by: ['alertname']
  group_wait: 5s
  group_interval: 5s
  repeat_interval: 5m
  receiver: 'email'
receivers:
- name: 'email'
  email_configs:
  - to: 'xxx@qq.com'
    send_resolved: true
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

**注: smtp_from、smtp_auth_username、to的邮箱可以填写同一个，smtp_auth_password填写鉴权码，需要开启POS3。**

如果不知道怎么开启POS3，可以查看我的这篇文章: [https://www.cnblogs.com/xuwujing/p/10945698.html](https://www.cnblogs.com/xuwujing/p/10945698.html)

------------


Prometheus服务的Prometheus.yml配置如下:

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets: 
      - '192.168.214.129:9093'
# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
   - "/opt/prometheus/prometheus-2.19.3.linux-amd64/first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['192.168.214.129:9090']

  - job_name: 'server'
    static_configs:
    - targets: ['192.168.214.129:9100']
```

**注：targets如果有多个配置的话，在后面加上其他服务的节点即可。alertmanagers最好写服务器的ip，不然可能会出现告警数据无法发送的情况。**

```bash
Error sending alert" err="Post \"http://alertmanager:9093/api/v1/alerts\": context deadline exceeded
```


----------


配置了Prometheus.yml之后，我们还需要配置告警的规则，也就是触发条件，达到条件之后就进行触发。我们新建一个first_rules.yml，用于检测服务器挂掉的时候进行发送消息

first_rules.yml告警配置:

**注:job等于的服务名称填写Prometheus.yml配置对应的名称，比如这里设置的server对应Prometheus.yml配置的server。**

```yaml
groups:
- name: node
  rules:
  - alert: server_status
    expr: up{job="server"} == 0 
    for: 15s
    annotations:
      summary: "机器{{ $labels.instance }} 挂了"
      description: "报告.请立即查看!"
```




----------


依次启动prometheus、altermanagers、node_server服务，查看告警，然后停止node_export服务，等待一段时间在查看。

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121434858-420444986.png)  
![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121435082-459383591.png)

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121435310-710214653.png)




# 2 Prometheus 监控 springboot 并且 触发Alert



Prometheus 监控应用的方式非常简单，只需要进程暴露了一个用于获取当前监控样本数据的 HTTP 访问地址。这样的一个程序称为Exporter，Exporter 的实例称为一个 Target 。Prometheus通过轮训的方式定时从这些 Target 中获取监控数据样本，对于应用来讲，只需要暴露一个包含监控数据的 HTTP访问地址即可，当然提供的数据需要满足一定的格式，这个格式就是 Metrics 格式: metric name>{=, ...} 。label name是标签,label value是标签的值



**Springboot应用实现步骤**  
1.在pom文件添加

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

2.在代码中添加如下配置：

```java
private Counter requestErrorCount;
    private final MeterRegistry registry;
    @Autowired
    public PrometheusCustomMonitor(MeterRegistry registry) {
        this.registry = registry;
    }
    @PostConstruct
    private void init() {
        requestErrorCount = registry.counter("requests_error_total", "status", "error");
    }
    public Counter getRequestErrorCount() {
        return requestErrorCount;
    }
```

3.在异常处理中添加如下记录:

```scss
monitor.getRequestErrorCount().increment();
```

4.在prometheus的配置中添加springboot应用服务监控

```yaml
-  job_name: 'springboot' 
     metrics_path: '/actuator/prometheus' 
     scrape_interval: 5s
     static_configs:
     - targets: ['192.168.8.45:8080'] 
```

5.Prometheu.yml配置如下:

```yaml
  - job_name: 'springboot' 
    metrics_path: '/actuator/prometheus' 
    scrape_interval: 5s
    static_configs:
    - targets: ['192.168.8.45:8080']  
```

规则文件配置如下:  
![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121437892-962827075.png)

6.在prometheus监控即可查看

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121438155-1857390664.png)  
企业微信告警效果图:  
![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121438479-1539076556.png)  
监控的springboot项目地址:[https://github.com/xuwujing/springBoot-study](https://github.com/xuwujing/springBoot-study)


# 3 一些告警配置

这是本人整理的一些服务应用告警的配置，也欢迎大家共同讨论一些常用的相关配置。

## 3.1 内存告警设置

```yaml
- name: test-rule
  rules:
  - alert: "内存报警"
    expr: 100 - ((node_memory_MemAvailable_bytes * 100) / node_memory_MemTotal_bytes) > 30
    for: 15s
    labels:
      severity: warning
    annotations:
      summary: "服务名:{{$labels.instance}}内存使用率超过30%了"
      description: "业务500报警: {{ $value }}"
      value: "{{ $value }}"
```

示例图:

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121438940-369972765.png)

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121439180-893674689.png)

## 3.2 磁盘设置

总量百分比设置:

```fsharp
(node_filesystem_size_bytes {mountpoint ="/"} - node_filesystem_free_bytes {mountpoint ="/"}) / node_filesystem_size_bytes {mountpoint ="/"} * 100
```

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121439485-1190290058.png)

---


查看某一目录的磁盘使用百分比

```fsharp
(node_filesystem_size_bytes{mountpoint="/boot"}-node_filesystem_free_bytes{mountpoint="/boot"})/node_filesystem_size_bytes{mountpoint="/boot"} * 100
```

---

正则表达式来匹配多个挂载点

> (node_filesystem_size_bytes{mountpoint="/|/run"}-node_filesystem_free_bytes{mountpoint="/|/run"})  
> / node_filesystem_size_bytes{mountpoint=~"/|/run"} * 100


----

预计多长时间磁盘爆满

> predict_linear(node_filesystem_free_bytes {mountpoint ="/"}[1h],  
> 4_3600) < 0 predict_linear(node_filesystem_free_bytes  
> {job="node"}[1h], 4_3600) < 0

---

CPU使用率

> 100 - (avg(irate(node_cpu_seconds_total{mode="idle"}[5m])) by  
> (instance) * 100)


---


空闲内存剩余率

> (node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes)  
> / node_memory_MemTotal_bytes * 100

---


内存使用率

> 100 -  
> (node_memory_MemFree_bytes+node_memory_Cached_bytes+node_memory_Buffers_bytes)  
> / node_memory_MemTotal_bytes * 100


---


磁盘使用率

> 100 - (node_filesystem_free_bytes{mountpoint="/",fstype=~"ext4|xfs"} /  
> node_filesystem_size_bytes{mountpoint="/",fstype=~"ext4|xfs"} * 100)



