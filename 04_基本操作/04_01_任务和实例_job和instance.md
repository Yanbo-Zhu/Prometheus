

在上一小节中，通过在prometheus.yml配置文件中，添加如下配置。我们让Prometheus可以从node exporter暴露的服务中获取监控指标数据。

```
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```

# 1 instance 

>- 用Prometheus术语来说，可以抓取的端点称为实例 instance



当我们需要采集不同的监控指标(例如：主机、MySQL、Nginx)时，我们只需要运行相应的监控采集程序，并且让Prometheus Server知道这些Exporter实例的访问地址。在Prometheus中，每一个暴露监控样本数据的HTTP服务称为一个实例。例如在当前主机上运行的node exporter可以被称为一个实例(Instance)。



# 2 job

>  具有相同目的的实例的集合（例如，出于可伸缩性或可靠性而复制的过程）称为job


而一组用于相同采集目的的实例，或者同一个采集进程的多个副本则通过一个一个任务(Job)进行管理。

```
* job: node
    * instance 2: 1.2.3.4:9100
    * instance 4: 5.6.7.8:9100
```


当前在每一个Job中主要使用了静态配置(static_configs)的方式定义监控目标。除了静态配置每一个Job的采集Instance地址以外，Prometheus还支持与DNS、Consul、E2C、Kubernetes等进行集成实现自动发现Instance实例，并从这些Instance上获取监控数据。

```
举例
  - job_name: 'pushgateway'
    honor_timestamps: true
    scrape_interval: 15s
    scrape_timeout: 10s
    metrics_path: /metrics
    scheme: http
    static_configs:
    - targets:
      - 172.20.70.205:9091
      - 172.20.70.205:9092
      - 172.20.70.215:9091
```



# 3 自动生成的标签和时间序列

当Prometheus抓取目标时，它会自动在抓取的时间序列上附加一些标签，以识别被抓取的目标：

```
- job：目标所属的已配置作业名称。
- instance：<host>:<port>抓取的目标网址的一部分。
```



# 4 prometheus特殊tag说明


```
- __address__ 采集endpoint的地址
- __name__   metrics 的名称
- instance   endpoint最后的tag
- job         任务
- __metrics_path__  采集的http path 如 /metrics  /cadvisor/metrics 
```

# 5 例子 

## 5.1 例子1 

- `up{job="<job-name>", instance="<instance-id>"}`：1实例是否正常（即可达）或0刮取失败。
设置告警查看采集失败的实例 `up==0`

## 5.2 例子2 

`scrape_duration_seconds{job="<job-name>", instance="<instance-id>"}`：刮擦的耗时

```
scrape_duration_seconds{instance="172.20.70.205", job="blackbox-ssh"} 0.001817932
scrape_duration_seconds{instance="172.20.70.205:3000", job="single-targets"} 0.005416658
scrape_duration_seconds{instance="172.20.70.205:9091", job="pushgateway"} 0.002726714
scrape_duration_seconds{instance="172.20.70.205:9092", job="pushgateway"} 0.000506256
scrape_duration_seconds{instance="172.20.70.205:9100", job="single-targets"} 0.012790691
scrape_duration_seconds{instance="172.20.70.205:9104", job="single-targets"} 0.021421043
scrape_duration_seconds{instance="172.20.70.205:9115", job="blackbox-http-targets"} 0.00427973
```

用途：统计job中采集比较耗时的instance ,


- 为什么慢
  - 网络质量
  - metrics数据量太大
  - prometheus采集端有瓶颈了，需要扩容


上次采集最慢的五个 job+instance topk(5,scrape_duration_seconds)
- 采集时间超过3秒的 scrape_duration_seconds > 3


## 5.3 例子3 

`scrape_samples_post_metric_relabeling{job="<job-name>", instance="<instance-id>"}`：relabel之后剩余的重新标记后剩余的样本数
- 何为样本：简单理解就是 标签组唯一 

## 5.4 例子4 

` scrape_samples_scraped{job="<job-name>", instance="<instance-id>"}`：目标暴露的样本数

```
scrape_samples_scraped{instance="172.20.70.205:9256", job="single-targets"} 1691
scrape_samples_scraped{instance="172.20.70.215:9256", job="single-targets"} 1010
scrape_samples_scraped{instance="172.20.70.205:9104", job="single-targets"} 816
scrape_samples_scraped{instance="172.20.70.215:9100", job="single-targets"} 500
scrape_samples_scraped{instance="172.20.70.205:9100", job="single-targets"} 500
```


用途： 统计样本数量按 job+instance分类
按job排序 topk(5,sum(scrape_samples_scraped) by (job))

```

{job="single-targets"} 4957
{job="redis_exporter_targets"} 299
{job="pushgateway"} 102
{job="blackbox-http-targets"} 72
{job="blackbox-ssh"} 6

```

## 5.5 例子5 
 
 `scrape_series_added{job="<job-name>", instance="<instance-id>"}`：此抓取中新系列的大概数量。v2.10的新功能
  - 用途 统计新增的metrics，可以用来查看写峰
  - 大部分情况应该都是旧的metrics append写入




# 6 UI 上看 

除了通过使用“up”表达式查询当前所有Instance的状态以外，还可以通过Prometheus UI中的Targets页面查看当前所有的监控采集任务，以及各个任务下所有实例的状态:

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPS8BVjkRvEjV8HmbBi%252F-LPS8D3XxcvpXC77SD3b%252Fprometheus_ui_targets.png%3Fgeneration%3D1540234733488865%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=d9425f80&sv=1)



我们也可以访问[http://192.168.33.10:9090/targets](http://192.168.33.10:9090/targets)直接从Prometheus的UI中查看当前所有的任务以及每个任务对应的实例信息。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPS8BVjkRvEjV8HmbBi%252F-LPS8D3ZspZkWr1CZto5%252Fprometheus_ui_targets_status.png%3Fgeneration%3D1540234733638602%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=2961c514&sv=1)


