
使用Node Exporter采集主机数据

# 1 安装Node Exporter

在Prometheus的架构设计中，Prometheus Server并不直接服务监控特定的目标，其主要任务负责数据的收集，存储并且对外提供数据查询支持。因此为了能够能够监控到某些东西，如主机的CPU使用率，我们需要使用到Exporter。Prometheus周期性的从Exporter暴露的HTTP服务地址（通常是/metrics）拉取监控样本数据。

从上面的描述中可以看出Exporter可以是一个相对开放的概念，其可以是一个独立运行的程序独立于监控目标以外，也可以是直接内置在监控目标中。只要能够向Prometheus提供标准格式的监控样本数据即可。

这里为了能够采集到主机的运行指标如CPU, 内存，磁盘等信息。我们可以使用[Node Exporter](https://github.com/prometheus/node_exporter)。


## 1.1 二进制安装 

Node Exporter同样采用Golang编写，并且不存在任何的第三方依赖，只需要下载，解压即可运行。可以从[https://prometheus.io/download/](https://prometheus.io/download/)获取最新的node exporter版本的二进制包。

```
curl -OL https://github.com/prometheus/node_exporter/releases/download/v0.15.2/node_exporter-0.15.2.darwin-amd64.tar.gz
tar -xzf node_exporter-0.15.2.darwin-amd64.tar.gz
```
然后移动到/opt/prometheus文件夹里面，没有该文件夹则创建



## 1.2 Docker安装

使用Docker运行，测试在Linux可以，在Mac不行：

docker run --rm --net=host --pid=host -v "/:/host:ro,rslave" \
  prom/node-exporter --path.rootfs=/host


在Darwin下，还是直接运行二进制比较好，注意还是需要允许运行会提示是恶意软件：
./node_exporter-1.3.1.darwin-amd64/node_exporter


Darwin下测试用，也可以直接非host方式运行：
docker run --rm -p 9100:9100 \


## 1.3 Docker Compose 方式去安装 

1
目录准备
创建目录
```
mkdir -pv /apps/exporter/node-exporter
```


2 编辑 docker-compose.yml 文件
```
vim /apps/exporter/node-exporter/docker-compose.yaml


version: "3"
	services:
	  node-exporter:
	    image: prom/node-exporter:v1.3.1
	    container_name: prometheus-node-exporter
	    hostname: node-exporter
	    restart: always
	    ports:
	      - 9100:9100
	networks:
	  default:
	    external: 
	      name: prometheus

```



3 创建 docker 网段 prometheus

```
检查是否存在 prometheus 网段：
docker network list

若不存在，则创建：
docker network create prometheus --subnet 10.21.22.0/24
```



# 2 启动 node_exporter的方式 

## 2.1 docker的方式启动

启动服务：
docker run --rm -it -v `pwd`/prometheus.yml:/etc/prometheus/prometheus.yml \
  -p 9090:9090 prom/prometheus


## 2.2 直接启动

运行node exporter:

```
cd node_exporter-0.15.2.darwin-amd64
cp node_exporter-0.15.2.darwin-amd64/node_exporter /usr/local/bin/
node_exporter
```


启动 node_exporter的方式2
root用户下启动
输入:
```bash
nohup ./consul_exporter   >/dev/null   2>&1 &
```



启动成功后，可以看到以下输出：

```
INFO[0000] Listening on :9100                            source="node_exporter.go:76"
```

访问[http://localhost:9100/](http://localhost:9100/)可以看到以下页面：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPMFlGDFIX7wuLhSHx9%252F-LPMFp6oc_SZOU4__NeX%252Fnode_exporter_home_page.png%3Fgeneration%3D1540136067584405%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=c5432eb&sv=1)


```
[root@node00 node_exporter]# ./node_exporter 
INFO[0000] Starting node_exporter (version=0.18.1, branch=HEAD, revision=3db77732e925c08f675d7404a8c46466b2ece83e)  source="node_exporter.go:156"
INFO[0000] Build context (go=go1.12.5, user=root@b50852a1acba, date=20190604-16:41:18)  source="node_exporter.go:157"
INFO[0000] Enabled collectors:                           source="node_exporter.go:97"
# 中间输出省略
INFO[0000] Listening on :9100                            source="node_exporter.go:170"

复制代码
```


## 2.3 docker-compose的方式去启动

4 
运行 node-exporter 容器
```
cd /apps/exporter/node-exporter
docker-compose up -d
```


5 检查  node-exporter 容器状态、查看  node-exporter 容器日志
```
cd /apps/exporter/node-exporter
docker-compose ps
docker-compose logs -f
```


6 重启 prometheus
cd /apps/prometheus
docker-compose restart
检查 node-exporter 数据是否正常上报
访问 Prometheus WebUI 的 targets 页面，检查 job 的状态




# 3 编译执行

make
./node_exporter


# 4 测试node_exporter

```
[root@node00 ~]# curl 127.0.0.1:9100/metrics<br># 这里可以看到node_exporter暴露出来的数据。
```



# 5 配置node_exporter开机自启


```
[root@node00 system]# cd /usr/lib/systemd/system# 准备systemd文件
[root@node00 systemd]# cat node_exporter.service 
[Unit]
Description=node_exporter
After=network.target 

[Service]
User=prometheus
Group=prometheus
ExecStart=/usr/local/exporter/node_exporter/node_exporter\
          --web.listen-address=:20001\
          --collector.systemd\
          --collector.systemd.unit-whitelist=(sshd|nginx).service\
          --collector.processes\
          --collector.tcpstat\
          --collector.supervisord
[Install]
WantedBy=multi-user.target
# 启动
[root@node00 exporter]# systemctl restart node_exporter# 查看状态
[root@node00 exporter]# systemctl status node_exporter
● node_exporter.service - node_exporter
   Loaded: loaded (/usr/lib/systemd/system/node_exporter.service; enabled; vendor preset: disabled)
   Active: active (running) since Fri 2019-09-20 22:43:09 EDT; 5s ago
 Main PID: 88262 (node_exporter)
   CGroup: /system.slice/node_exporter.service
           └─88262 /usr/local/exporter/node_exporter/node_exporter --collector.systemd --collector.systemd.unit-whitelist=(sshd|nginx).service

Sep 20 22:43:09 node00 node_exporter[88262]: time="2019-09-20T22:43:09-04:00" level=info msg=" - stat" source="node_exporter.go:104"
Sep 20 22:43:09 node00 node_exporter[88262]: time="2019-09-20T22:43:09-04:00" level=info msg=" - systemd" source="node_exporter.go:104"
Sep 20 22:43:09 node00 node_exporter[88262]: time="2019-09-20T22:43:09-04:00" level=info msg=" - textfile" source="node_exporter.go:104"
Sep 20 22:43:09 node00 node_exporter[88262]: time="2019-09-20T22:43:09-04:00" level=info msg=" - time" source="node_exporter.go:104"
Sep 20 22:43:09 node00 node_exporter[88262]: time="2019-09-20T22:43:09-04:00" level=info msg=" - timex" source="node_exporter.go:104"
Sep 20 22:43:09 node00 node_exporter[88262]: time="2019-09-20T22:43:09-04:00" level=info msg=" - uname" source="node_exporter.go:104"
Sep 20 22:43:09 node00 node_exporter[88262]: time="2019-09-20T22:43:09-04:00" level=info msg=" - vmstat" source="node_exporter.go:104"
Sep 20 22:43:09 node00 node_exporter[88262]: time="2019-09-20T22:43:09-04:00" level=info msg=" - xfs" source="node_exporter.go:104"
Sep 20 22:43:09 node00 node_exporter[88262]: time="2019-09-20T22:43:09-04:00" level=info msg=" - zfs" source="node_exporter.go:104"
Sep 20 22:43:09 node00 node_exporter[88262]: time="2019-09-20T22:43:09-04:00" level=info msg="Listening on :9100" source="node_exporter.go:170"# 开机自启[root@node00 exporter]# systemctl enable node_exporter
```



# 6 Prometheus配置文件prometheus.yml

编辑 prometheus.yml 文件


```yaml
scrape_configs:
  - job_name: "node"
    metrics_path: "/metrics"
    scrape_interval: 5s
    static_configs:
      - targets: ["host.docker.internal:9100"]
```


```
[root@node00 prometheus]# cd /usr/local/prometheus/prometheus
[root@node00 prometheus]# vim prometheus.yml
# 在scrape_configs中加入job node ,最终scrape_configs如下配置
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: "node"
    static_configs:
    - targets: 
      - "192.168.100.10:20001"

[root@node00 prometheus]# systemctl restart prometheus 
[root@node00 prometheus]# systemctl status prometheus
```




# 7 初始Node Exporter监控指标

访问[http://localhost:9100/metrics](http://localhost:9100/metrics)，可以看到当前node exporter获取到的当前主机的所有监控数据，如下所示：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPMFlGDFIX7wuLhSHx9%252F-LPMFp6q8o3vUrTJOaFo%252Fnode_exporter_metrics_page.png%3Fgeneration%3D1540136066317078%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=b3d3ca24&sv=1)

主机监控指标

每一个监控指标之前都会有一段类似于如下形式的信息：

```
# HELP node_cpu Seconds the cpus spent in each mode.
# TYPE node_cpu counter
node_cpu{cpu="cpu0",mode="idle"} 362812.7890625
# HELP node_load1 1m load average.
# TYPE node_load1 gauge
node_load1 3.0703125
```

其中HELP用于解释当前指标的含义，TYPE则说明当前指标的数据类型。
在上面的例子中node_cpu的注释表明当前指标是cpu0上idle进程占用CPU的总时间，CPU占用时间是一个只增不减的度量指标，从类型中也可以看出node_cpu的数据类型是计数器(counter)，与该指标的实际含义一致。又例如node_load1该指标反映了当前主机在最近一分钟以内的负载情况，系统的负载情况会随系统资源的使用而变化，因此node_load1反映的是当前状态，数据可能增加也可能减少，从注释中可以看出当前指标类型为仪表盘(gauge)，与指标反映的实际含义一致。

除了这些以外，在当前页面中根据物理主机系统的不同，你还可能看到如下监控指标：
- node_boot_time：系统启动时间
- node_cpu：系统CPU使用量
- node_disk_*：磁盘IO
- node_filesystem_*：文件系统用量
- node_load1：系统负载
- node_memeory_*：内存使用量
- node_network_*：网络带宽
- node_time：当前系统时间
- go_*：node exporter中go相关指标
- process_*：node exporter自身进程相关运行指标


# 8 从Node Exporter收集监控数据

为了能够让Prometheus Server能够从当前node exporter获取到监控数据，这里需要修改Prometheus配置文件。编辑prometheus.yml并在scrape_configs节点下添加以下内容:

```
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  # 采集node exporter监控数据
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```

重新启动Prometheus Server

访问[http://localhost:9090](http://localhost:9090)，进入到Prometheus Server。如果输入“up”并且点击执行按钮以后，可以看到如下结果：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPMFlGDFIX7wuLhSHx9%252F-LPMFp6sd1HomUq2AhEt%252Fprometheus_ui_up_query.png%3Fgeneration%3D1540136068422883%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=48b6eec0&sv=1)

Expression Browser

如果Prometheus能够正常从node exporter获取数据，则会看到以下结果：

```
up{instance="localhost:9090",job="prometheus"}    1
up{instance="localhost:9100",job="node"}    1
```

其中“1”表示正常，反之“0”则为异常。


# 9 配置 Grafana 看板

登录 Grafana，导入对应的看板即可。
看板获取地址：https://grafana.com/grafana/dashboards/?dataSource=prometheus&collector=nodeexporter

注意： 看板导入后需要修改数据源的ID

数据源查看方式： 在 Grafana 中进入 数据源详情 页面，浏览器 URL 的最后一段字符为该数据源的 ID。 如 URL 为 grafana/datasources/edit/6lbJpCb4z 时， 6lbJpCb4z 即为当前数据源的 ID

数据源替换方式： 编辑看板，查看看板的 JSON 数据，替换 datasource 中的 uid

