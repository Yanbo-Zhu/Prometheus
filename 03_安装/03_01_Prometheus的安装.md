
https://hty1024.com/archives/prometheus-jian-kong-fang-an-xue-xi-bi-ji--er-prometheusserver-he-grafana-de-an-zhuang-he-pei-zhi

Prometheus基于Golang编写，编译后的软件包，不依赖于任何的第三方依赖。用户只需要下载对应平台的二进制包，解压并且添加基本的配置即可正常启动Prometheus Server。

# 1 从二进制包安装


1 
对于非Docker用户，可以从[https://prometheus.io/download/](https://prometheus.io/download/)找到最新版本的Prometheus Sevrer软件包：
下载地址：https://prometheus.io/download

```
export VERSION=2.4.3
curl -LO  https://github.com/prometheus/prometheus/releases/download/v$VERSION/prometheus-$VERSION.darwin-amd64.tar.gz
```


```
cd /apps/prometheus/2.33.1
wget https://github.com/prometheus/prometheus/releases/download/v2.33.1/prometheus-2.33.1.linux-amd64.tar.gz
```


2 

解压安装包
```
cd /apps/prometheus/2.33.1
tar -xzf prometheus-2.33.1.linux-amd64.tar.gz --strip-components 1
```

解压，并将Prometheus相关的命令，添加到系统环境变量路径即可：
```
tar -xzf prometheus-${VERSION}.darwin-amd64.tar.gz
cd prometheus-${VERSION}.darwin-amd64
```

然后移动到/opt/prometheus文件夹里面，没有该文件夹则创建

解压后当前目录会包含默认的Prometheus配置文件promethes.yml:
```
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
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
```



3 
Promtheus作为一个时间序列数据库，其采集的数据会以文件的形式存储在本地中，默认的存储路径为`data/`，因此我们需要先手动创建该目录：
```
mkdir -p data
```

用户也可以通过参数`--storage.tsdb.path="data/"`修改本地数据存储的路径。

----


创建存储目录
```
mkdir -p /apps/prometheus/2.33.1/data
```


配置相关参数
编辑 prometheus.yml 文件，内容如下：

```yaml
# my global config
	global:
	  scrape_interval:    15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
	  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
	  # scrape_timeout is set to the global default (10s).
	 
	# Alertmanager configuration
	alerting:
	  alertmanagers:
	  - static_configs:
	   - targets:
	     # - alertmanager:9093
	 
	# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
	rule_files:
	  # - "first_rules.yml"
	  # - "second_rules.yml"
	 
	# A scrape configuration containing exactly one endpoint to scrape:
	# Here it's Prometheus itself.
	scrape_configs:
	  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
	  - job_name: 'prometheus'
	 
	   # metrics_path defaults to '/metrics'
	   # scheme defaults to 'http'.
	 
	   static_configs:
	   - targets: ['localhost:9090']
```


## 1.1 prometheus 启动
启动prometheus服务，其会默认加载当前路径下的prometheus.yaml文件：
在/opt/prometheus/prometheus-2.19.3.linux-amd64的目录下输入:

```bash
# 前台运行
nohup ./prometheus   >/dev/null   2>&1 &
./prometheus

./prometheus


# 后台运行
./prometheus --config.file=prometheus.yml  --web.enable-lifecycle > /apps/prometheus/2.33.1/logs/prometheus.log 2>&1 &

```



启动成功之后，在浏览器上输入 ip+9090可以查看相关信息。
![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121425628-625987805.png)  
![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121425898-2021261055.png)



正常的情况下，你可以看到以下输出内容：

```
level=info ts=2018-10-23T14:55:14.499484Z caller=main.go:554 msg="Starting TSDB ..."
level=info ts=2018-10-23T14:55:14.499531Z caller=web.go:397 component=web msg="Start listening for connections" address=0.0.0.0:9090
level=info ts=2018-10-23T14:55:14.507999Z caller=main.go:564 msg="TSDB started"
level=info ts=2018-10-23T14:55:14.508068Z caller=main.go:624 msg="Loading configuration file" filename=prometheus.yml
level=info ts=2018-10-23T14:55:14.509509Z caller=main.go:650 msg="Completed loading of configuration file" filename=prometheus.yml
level=info ts=2018-10-23T14:55:14.509537Z caller=main.go:523 msg="Server is ready to receive web requests."
```




# 2 使用容器安装

对于Docker用户，直接使用Prometheus的镜像即可启动Prometheus Server：

```
docker pull prom/prometheus:2.33.1

docker run -p 9090:9090 -v /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
docker run -p 9090:9090 -v /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus

```


使用浏览器访问 Prometheus Server Web UI
浏览器访问 9090 端口即可

启动完成后，可以通过[http://localhost:9090](http://localhost:9090)访问Prometheus的UI界面：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPWdZfpgg-IxWhxja2D%252F-LPMFp6m_uMCLYKqaoBZ%252Fprometheus-ui-graph.png%3Fgeneration%3D1540310329044095%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=e3288e9c&sv=1)



## 2.1 直接本机Docker启动

直接本机Docker启动，然后访问本机后台：http://localhost:9090/
`docker run --name prometheus -d -p 127.0.0.1:9090:9090 prom/prometheus`



可以docker inspect看到它的信息：
配置文件：--config.file=/etc/prometheus/prometheus.yml
存储路径：--storage.tsdb.path=/prometheus

数据模型：
所有数据都是时序数据，time series。
时序数据可以有name和label来分类过滤，比如：


```
<metric name>{<label name>=<label value>, ...}
api_http_requests_total{method="POST", handler="/messages"}
```


# 3 使用 Docker-compose 安装（推荐）

```
目录准备, 创建相关目录：
mkdir pv /apps/prometheus/{conf,data,rules}

配置 data 目录权限：
chmod -R 777 /apps/prometheus/data/


编辑 docker-compose.yml 文件
vim /apps/prometheus/docker-compose.yml

version: "3"
	services:
	  prometheus:
	    image: prom/prometheus:v2.33.1
	    container_name: prometheus-prometheus
	    hostname: prometheus
	    restart: always
	    volumes:
	      - /apps/prometheus/data:/prometheus
	      - /apps/prometheus/rules:/etc/prometheus/rules
	      - /apps/prometheus/conf/prometheus.yml:/etc/prometheus/prometheus.yml
	    ports:
	      - 9090:9090
	    command: "--config.file=/etc/prometheus/prometheus.yml --web.enable-lifecycle"
	networks:
	  default:
	    external:
	      name: prometheus
```


注：
上方的 --web.enable-lifecycle 为启动热加载，添加后可以通过 curl http://服务器IP:PrometheusServer端口号/-/reload -X POST 重载配置文件


3 编辑 prometheus.yml 文件
vim /apps/prometheus/conf/prometheus.yml
```
# my global config
	global:
	  scrape_interval:     5s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
	  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
	 
	#alerting:
	#  alertmanagers:
	#  - static_configs:
	#    - targets: ['localhost:9093']
	 
	rule_files:
	  - "/etc/prometheus/rules/*.yml" 
	 
	scrape_configs:
	  - job_name: 'prometheus'
	    static_configs:
	      - targets: ['localhost:9090']
```



4 
创建 docker 网段 prometheus
检查是否已有prometheus 网段：

```
docker network list

若没有则创建：
docker network create prometheus --subnet 10.21.22.0/24

启动 prometheus 容器
cd /apps/prometheus
docker-compose up -d

检查prometheus 容器状态、查看 prometheus 容器日志
cd /apps/prometheus
docker-compose ps
docker-compose logs -f

```

访问 Prometheus Server Web UI
浏览器访问 9090 端口即可



# 4 获取配置帮助

```
[root@node00 prometheus]# ./prometheus  --help 
```



# 5 配置开机自启动_使用 systemctl 方式启动

```
vim /etc/systemd/system/prometheus.service

[Unit]
	Description=Prometheus Monitoring System
	Documentation=Prometheus Monitoring System
	After=network.target
	 
	[Service]
	#User=root
	#Group=root
	Type=simple
	# 启动脚本
	ExecStart=/apps/prometheus/2.33.1/prometheus --config.file=/apps/prometheus/2.33.1/prometheus.yml --web.enable-lifecycle
	ExecReload=/bin/kill -HUP $MAINPID
	Restart=on-failure
	 
	[Install]
	WantedBy=multi-user.target
	
```


```
# 重载
	systemctl daemon-reload

# 配置开机启动
	systemctl enable prometheus

# 启动
	systemctl start prometheus

# 查看状态
	systemctl status prometheus

# 停止
	systemctl stop prometheus

```

使用浏览器访问 Prometheus Server Web UI
浏览器访问 9090 端口访问即可

---


```sh

# 进入systemd文件目录[root@node00 system]# cd /usr/lib/systemd/system# 编写prometheus systemd文件
[root@node00 system]# cat prometheus.service 
[Unit]
Description=prometheus
After=network.target 

[Service]
User=prometheus
Group=prometheus
WorkingDirectory=/usr/local/prometheus/prometheus
ExecStart=/usr/local/prometheus/prometheus/prometheus
[Install]
WantedBy=multi-user.target


# 启动
[root@node00 system]# systemctl restart prometheus # 查看状态
[root@node00 system]# systemctl status prometheus
● prometheus.service - prometheus
   Loaded: loaded (/usr/lib/systemd/system/prometheus.service; disabled; vendor preset: disabled)
   Active: active (running) since Fri 2019-09-20 06:11:21 EDT; 4s ago
 Main PID: 32871 (prometheus)
   CGroup: /system.slice/prometheus.service
           └─32871 /usr/local/prometheus/prometheus/prometheus

Sep 20 06:11:21 node00 prometheus[32871]: level=info ts=2019-09-20T10:11:21.634Z caller=head.go:509 component=tsdb msg="replaying WAL, this may take awhile"
Sep 20 06:11:21 node00 prometheus[32871]: level=info ts=2019-09-20T10:11:21.640Z caller=head.go:557 component=tsdb msg="WAL segment loaded" segment=0 maxSegment=3
Sep 20 06:11:21 node00 prometheus[32871]: level=info ts=2019-09-20T10:11:21.640Z caller=head.go:557 component=tsdb msg="WAL segment loaded" segment=1 maxSegment=3
Sep 20 06:11:21 node00 prometheus[32871]: level=info ts=2019-09-20T10:11:21.641Z caller=head.go:557 component=tsdb msg="WAL segment loaded" segment=2 maxSegment=3
Sep 20 06:11:21 node00 prometheus[32871]: level=info ts=2019-09-20T10:11:21.641Z caller=head.go:557 component=tsdb msg="WAL segment loaded" segment=3 maxSegment=3
Sep 20 06:11:21 node00 prometheus[32871]: level=info ts=2019-09-20T10:11:21.642Z caller=main.go:669 fs_type=XFS_SUPER_MAGIC
Sep 20 06:11:21 node00 prometheus[32871]: level=info ts=2019-09-20T10:11:21.642Z caller=main.go:670 msg="TSDB started"
Sep 20 06:11:21 node00 prometheus[32871]: level=info ts=2019-09-20T10:11:21.642Z caller=main.go:740 msg="Loading configuration file" filename=prometheus.yml
Sep 20 06:11:21 node00 prometheus[32871]: level=info ts=2019-09-20T10:11:21.686Z caller=main.go:768 msg="Completed loading of configuration file" filename=prometheus.yml
Sep 20 06:11:21 node00 prometheus[32871]: level=info ts=2019-09-20T10:11:21.686Z caller=main.go:623 msg="Server is ready to receive web requests."# 开机自启配置
[root@node00 system]# systemctl enable prometheus
Created symlink from /etc/systemd/system/multi-user.target.wants/prometheus.service to /usr/lib/systemd/system/prometheus.service.
```








