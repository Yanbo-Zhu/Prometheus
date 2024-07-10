https://hty1024.com/archives/prometheus-jian-kong-fang-an-xue-xi-bi-ji--shi-er-prometheus-yi-jian-bu-shu-jiao-ben-shi-yong-shuo-ming


# 1 项目说明

项目描述： 基于 Ansible 的 Prometheus 监控组件一键部署脚本  
项目地址： [https://git.hty1024.com/hty1024/ansible_examples/-/tree/master/prometheus?ref_type=heads](https://git.hty1024.com/hty1024/ansible_examples/-/tree/master/prometheus?ref_type=heads)

  
# 2 注意事项

1. 此项目基于 Ansible，因此需要先安装 Ansible 并配置秘钥
2. 采用 Docker Compose 方式部署 Prometheus 相关组件，因此会自动安装 Docker 及 Docker Compse
3. 监控中间件的 Exporter 涉及配置项，建议参考官方文档或部署脚本手动部署
4. 若手动卸载后再次使用此脚本重装，则需要删除 ~/.flag/ 目录下的全部文件

  

# 3 使用说明

1. 安装 Ansible  
    略，详情请参考官网文档或本站文章（[Ansible（1）：Ansible的安装和配置](https://hty1024.com/ops/tools/ansible/1)）
    
2. 配置 Ansible 秘钥  
    略，详情请参考官网文档或本站文章（[Ansible（2）：常用命令及模块说明](https://hty1024.com/ops/tools/ansible/2)）
    
3. 创建相关目录
    

- Git 文件下载目录（此处示例使用 `/tmp/ansible`）

```bash
mkdir -pv /tmp/ansible
```

4. 克隆 Git 仓库文件  
    访问项目的 Git 地址（见上方），克隆文件至本地自定义目录

```bash
cd /tmp/ansible
```

```bash
git clone https://git.hty1024.com/hty1024/ansible_examples
```

5. 修改 Ansible 的 host 文件，指定安装主机

```bash
vim /etc/ansible/hosts
```

配置说明：

- **dockerce** ： 安装 Docker Engine 的主机
- **server** ： 安装 Prometheus Server 相关服务的主机，含：Prometheus Server、Grafana、Alertmanager、Prometheus Alert、Prometheus Webhook Dingtalk、Loki
- **node** ： 安装 Prometheus Node Exporter 的主机
- **docker** ： 安装 Prometheus CAdvisor 的主机
- **mysql** ： 安装 Prometheus Mysqld Exporter 的主机
- **mongodb** ： 安装 Prometheus Mongodb Exporter 的主机
- **redis** ： 安装 Prometheus Redis Exporter 的主机
- **rabbitmq** ： 安装 Prometheus Rabbitmq Exporter 的主机
- **rocketmq** ： 安装 Prometheus Rocketmq Exporter 的主机
- **elasticsearch** ： 安装 Prometheus Elasticsearch Exporter 的主机
- **zookeeper** ： 安装 Prometheus Zookeeper Exporter 的主机
- **jmx** ： 安装 Prometheus JMX Exporter 的主机
- **nginx** ： 安装 Nginx Prometheus Exporter 的主机
- **blackbox** ： 安装 Prometheus Blackbox Exporter 的主机
- **promtail** ： 安装 Grafana Promtail 的主机

配置示例（此处示例将 Prometheus 组件安装至 `172.31.0.201` 主机）：

```bash
[dockerce]
	172.31.0.201
	 
	[server]
	172.31.0.201
	 
	[node]
	172.31.0.201
	 
	[docker]
	172.31.0.201
	 
	[blackbox]
	172.31.0.201
	 
	[mysql]
	172.31.0.201
	 
	[mongodb]
	172.31.0.201
	 
	[redis]
	172.31.0.201
	 
	[rabbitmq]
	172.31.0.201
	 
	[rocketmq]
	172.31.0.201
	 
	[elasticsearch]
	172.31.0.201
	 
	[zookeeper]
	172.31.0.201
	 
	[jmx]
	172.31.0.201
	 
	[nginx]
	172.31.0.201
	 
	[promtail]
	172.31.0.201
```

5. 编辑安装配置文件

```bash
vim /tmp/ansible/ansible_examples/prometheus/roles/conf/extend.yml
```

可按需修改的地方：

- **base_dir**： Prometheus 监控组件部署目录
- **package**： 部署的 Docker 安装包名称（如需安装自定义版本的 Docker，则可自行从 Docker 官网下载对应的安装包，放置于 ansible_examples/prometheus/roles/docker/files/x86_64 或 aarch64 目录，然后此配置项为安装包名称）
- **subnet**： Prometheus 监控组件使用的网段，注意不要和主机网段冲突
- **各组件的 image**： Prometheus 监控组件使用的镜像，可按需修改 Tag
- **各组件的 port**： Prometheus 监控组件使用的端口号，可按需修改

文件示例：

```yaml
---
	# 部署目录（部署时会在此目录下生成各服务的子目录）
	base_dir: /opt
	 
	# Docker 配置
	docker: 
	  ## Docker Engine 安装包名称
	  package: docker-24.0.7.tgz
	  ## Docker 部署目录
	  dir:
	    ### 主目录
	    main: '{{ base_dir }}/docker'
	    ### 可执行文件目录
	    bin: '{{ base_dir }}/docker/bin'
	    ### 数据目录
	    data: '{{ base_dir }}/docker/data'
	  ## 部署 Prometheus 相关组件时使用的网络
	  network:
	    name: prometheus
	    subnet: 10.21.22.0/24
	 
	# Prometheus Server 配置
	prometheus:
	  ## 镜像
	  image: prom/prometheus:v2.48.0
	  ## 部署目录
	  dir:
	    ### 主目录
	    main: '{{ base_dir }}/prometheus'
	    ### 配置文件目录
	    conf: '{{ base_dir }}/prometheus/conf'
	    ### 告警规则文件目录
	    rules: '{{ base_dir }}/prometheus/rules'
	    ### 数据目录
	    data: '{{ base_dir }}/prometheus/data'
	  ## 端口号
	  port: 9090
	 
	# Grafana 配置
	grafana:
	  ## 是否部署 Grafana
	  enable: true
	  ## 镜像
	  image: grafana/grafana:10.2.2
	  ## 部署目录
	  dir:
	    ### 主目录
	    main: '{{ base_dir }}/grafana'
	    ### 配置文件目录
	    conf: '{{ base_dir }}/grafana/conf'
	    ### 数据目录
	    data: '{{ base_dir }}/grafana/data'
	  ## 端口号
	  port: 3000
	 
	# Alertmanager 配置
	alertmanager:
	  ## 是否部署 Alertmanager
	  enable: true
	  ## 镜像
	  image: prom/alertmanager:v0.26.0
	  ## 部署目录
	  dir:
	    ### 主目录
	    main: '{{ base_dir }}/alertmanager'
	    ### 配置文件目录
	    conf: '{{ base_dir }}/alertmanager/conf'
	    ### 模板目录
	    template: '{{ base_dir }}/alertmanager/template'
	  ## 端口号
	  port: 9093
	 
	# Prometheus Alert 配置 （ Prometheus Alert 和 Prometheus Webhook Dingtalk 只能开启其中一个，不能同时开启）
	prometheus_alert:
	  ## 是否启用 Prometheus Alert
	  enable: true
	  ## 镜像
	  image: feiyu563/prometheus-alert:v4.9
	  ## 部署目录
	  dir: 
	    ### 主目录
	    main: '{{ base_dir }}/prometheus-alert'
	    ### 配置文件目录
	    conf: '{{ base_dir }}/prometheus-alert/conf'
	    ### 数据库目录
	    db: '{{ base_dir }}/prometheus-alert/db'
	    ### 数据文件目录
	    logs: '{{ base_dir }}/prometheus-alert/logs'
	  ## 端口号
	  port: 9080 #8080
	  ## 用户名
	  user: 'admin'
	  ## 密码
	  password: '123456'
	  ## 标题
	  title: 'PrometheusAlert'
	  ## 访问 URL
	  url: 'https://alert.hty1024.com'
	 
	# Prometheus Webhook Dingtalk 配置 （ Prometheus Alert 和 Prometheus Webhook Dingtalk 只能开启其中一个，不能同时开启）
	prometheus_webhook_dingtalk:
	  ## 是否启用 Prometheus Webhook Dingtalk
	  enable: false
	  ## 镜像
	  image: timonwong/prometheus-webhook-dingtalk:v2.1.0
	  ## 部署目录
	  dir:
	    ### 主目录
	    main: '{{ base_dir }}/prometheus-webhook-dingtalk'
	    ### 配置文件目录
	    conf: '{{ base_dir }}/prometheus-webhook-dingtalk/conf'
	    ### 模板文件目录
	    template: '{{ base_dir }}/prometheus-webhook-dingtalk/template'
	  ## 端口号
	  port: 9060 #8060
	 
	# 告警配置
	alert:
	  ## 钉钉机器人
	  dingtalk:
	   access_token: '123456789'
	  ## 短信
	  message:
	    type: 'tx'
	    phone: '12300001111'
	    ### 注意：部署完成之后，需要手动将 prometheus-webhook-dingtalk/conf/config.yml 文件的 “【【” 替换为 “{{”，将 “】】” 替换为 “}}”
	    title: '【【 template "legacy.title" . 】】'
	    text: '【【 template "email.to.message" . 】】'
	 
	# Loki 配置
	loki:
	  ## 是否启用 Loki
	  enable: true
	  ## 镜像
	  image: grafana/loki:2.9.2
	  ## 部署目录
	  dir:
	    ### 主目录
	    main: '{{ base_dir }}/loki'
	    ### 配置文件目录
	    conf: '{{ base_dir }}/loki/conf'
	  ## 端口号
	  port: 3100
	 
	# Node Exporter 配置
	node_exporter:
	  ## 是否启用 Node Exporter 
	  enable: true
	  ## 镜像
	  image: prom/node-exporter:v1.7.0
	  ## 部署目录
	  dir: '{{ base_dir }}/exporter/node-exporter'
	  ## 端口号
	  port: 9100
	 
	# CAdvisor 配置
	cadvisor:
	  ## 是否启用 CAdvisor
	  enable: true
	  ## 镜像
	  image: zcube/cadvisor:v0.45.0
	  ## 部署目录
	  dir: '{{ base_dir }}/exporter/cadvisor'
	  ## 端口号
	  port: 9180 #8088
	 
	# Blackbox Exporter 配置
	blackbox_exporter:
	  ## 是否启用 Blackbox Exporter
	  enable: true
	  ## 镜像
	  image: prom/blackbox-exporter:v0.24.0
	  ## 部署目录
	  dir:
	    ### 主目录
	    main: '{{ base_dir }}/exporter/blackbox'
	    ### 配置文件目录
	    conf: '{{ base_dir }}/exporter/blackbox/conf'
	  ## 端口号
	  port: 9115
	  ## 探测 URL
	  targets:
	    - https://hty1024.com
	    - https://www.hty1024.com
	    - https://wiki.hty1024.com
	    - https://git.hty1024.com
	 
	# Mysqld Exporter 配置
	mysqld_exporter:
	  ## 镜像
	  image: prom/mysqld-exporter:v0.15.0
	  ## 部署目录
	  dir: '{{ base_dir }}/exporter/mysqld-exporter'
	  ## 采集节点
	  nodes:
	      ### 主机
	    - host: 127.0.0.1
	      ### Mysqld Exporter 端口号
	      port: 9104
	      ### MySQL URL
	      url: 127.0.0.1:3306
	      ### MySQL 用户名
	      user: exporter
	      ### MySQL 密码
	      password: 123456
	      ### MySQL 数据库
	      database: 
	    - host: 192.168.1.1
	      port: 9104
	      url: 192.168.1.1
	      user: exporter
	      password: 123456
	      database: 
	 
	# MongoDB Exporter 配置
	mongodb_exporter:
	  ## 镜像
	  image: bitnami/mongodb-exporter:0.40.0
	  ## 部署目录
	  dir: '{{ base_dir }}/exporter/mongodb-exporter'
	  ## 采集节点
	  nodes:
	      ### 主机
	    - host: 127.0.0.1
	      ### MongoDB Exporter 端口号
	      port: 9126
	      ### MongoDB URL
	      url: 127.0.0.1:27017
	    - host: 192.168.1.1
	      port: 9126
	      url: 192.168.1.1:27017
	 
	# Redis Exporter 配置
	redis_exporter:
	  ## 镜像
	  image: oliver006/redis_exporter:v1.55.0
	  ## 部署目录
	  dir: '{{ base_dir }}/exporter/redis-exporter'
	  ## 采集节点
	  nodes:
	      ### 主机
	    - host: 127.0.0.1
	      ### Redis Exporter 端口号
	      port: 9121
	      ### Redis URL
	      url: 127.0.0.1:6379
	      ### Redis 密码
	      password: 123456
	    - host: 192.168.1.1
	      port: 9121
	      url: 127.0.0.1:6379
	      password: 123456
	 
	# Rabbitmq Exporter 配置
	rabbitmq_exporter:
	  ## 镜像
	  image: kbudde/rabbitmq-exporter:1.0.0-RC19
	  ## 部署目录
	  dir: '{{ base_dir }}/exporter/rabbitmq-exporter'
	  ## 采集节点
	  nodes:
	      ### 主机
	    - host: 127.0.0.1
	      ### Rabbitmq Exporter 端口号
	      port: 9419
	      ### RabbitMQ URL
	      url: 127.0.0.1:15672
	      ### RabbitMQ 用户名
	      user: root
	      ### RabbitMQ 密码
	      password: 123456
	    - host: 192.168.1.1
	      port: 9419
	      url: 127.0.0.1:15672
	      user: root
	      password: 123456
	 
	# Rocketmq Exporter 配置
	rocketmq_exporter:
	  ## 镜像
	  image: bitnami/java:1.8.392-9
	  ## jar 包名称
	  jar: rocketmq-exporter-0.0.2-SNAPSHOT.jar
	  ## 部署目录
	  dir: '{{ base_dir }}/exporter/rocketmq-exporter'
	  ## 采集节点
	  nodes:
	      ### 主机
	    - host: 127.0.0.1
	      ### Rocketmq Exporter 端口号
	      port: 5557
	      ### Rocketmq URL
	      url: 127.0.0.1:9876
	      ### Rocketmq 版本号
	      version: V4_9_4
	    - host: 192.168.1.1
	      port: 5557
	      url: 127.0.0.1:9876
	      version: V4_9_4
	 
	# Elasticsearch Exporter 配置
	elasticsearch_exporter:
	  ## 镜像
	  image: prometheuscommunity/elasticsearch-exporter:v1.6.0
	  ## 部署目录
	  dir: '{{ base_dir }}/exporter/elasticsearch-exporter'
	  ## 采集节点
	  nodes:
	      ### 主机
	    - host: 127.0.0.1
	      ### Elasticsearch Exporter 端口号
	      port: 9114
	      ### Elasticsearch URL
	      url: 127.0.0.1:9200
	    - host: 192.168.1.1
	      port: 9114
	      url: 127.0.0.1:9200
	 
	# Zookeeper Exporter 配置
	zookeeper_exporter:
	  ## 镜像
	  image: dabealu/zookeeper-exporter:v0.1.13
	  ## 部署目录
	  dir: '{{ base_dir }}/exporter/zookeeper-exporter'
	  ## 采集节点
	  nodes:
	      ### 主机
	    - host: 127.0.0.1
	      ### Zookeeper Exporter 端口号
	      port: 9141
	      ### Zookeeper URL
	      url: 127.0.0.1:2181
	    - host: 192.168.1.1
	      port: 9141
	      url: 127.0.0.1:2181
	 
	# JMX Exporter 配置
	jmx_exporter:
	  ## 运行模式（ javaagent：Java Agent 模式运行；httpserver：HTTP Server 模式运行）
	  model: httpserver
	  ## 镜像
	  image: bitnami/java:1.8.392-9
	  ## jar 包名称
	  jar: jmx_prometheus_httpserver-0.17.2.jar
	  ## 部署目录
	  dir: 
	    ### 主目录
	    main: '{{ base_dir }}/exporter/jmx-exporter'
	    ### 配置文件目录
	    conf: '{{ base_dir }}/exporter/jmx-exporter/conf'
	  ## 采集节点
	  nodes:
	      ### 主机
	    - host: 127.0.0.1
	      ### JMX Exporter 端口号
	      port: 9151
	      ### JMX Exporter URL
	      url: localhost:8080
	    - host: 192.168.1.1
	      port: 9151
	      url: localhost:8080
	 
	# Nginx Prometheus Exporter 配置
	nginx_prometheus_exporter:
	  ## 镜像
	  image: nginx/nginx-prometheus-exporter:0.11.0
	  ## 部署目录
	  dir: '{{ base_dir }}/exporter/nginx-prometheus-exporter'
	  ## 采集节点
	  nodes:
	      ### 主机
	    - host: 127.0.0.1
	      ### Nginx Prometheus Exporter 端口号
	      port: 9113
	      ### Nginx URL
	      url: http://127.0.0.1/nginx_status
	    - host: 192.168.1.1
	      port: 9113
	      url: http://127.0.0.1/nginx_status
	 
	# Promtail 配置
	promtail:
	  ## 镜像
	  image: grafana/promtail:2.9.2
	  ## 部署目录
	  dir:
	    ### 主目录
	    main: '{{ base_dir }}/exporter/promtail'
	    ### 配置文件目录
	    conf: '{{ base_dir }}/exporter/promtail/conf'
	  ## 采集节点
	  nodes:
	      ### 主机
	    - host: 127.0.0.1
	      ### Promtail 端口号
	      port: 9188
	      ### 需要收集的日志目录
	      targets:
	          #### 名称
	        - name: system
	          #### 路径
	          path: /var/log 
	    - host: 192.168.1.1
	      port: 9188
	      targets: 
	        - name: system
	          path: /var/log
```

6. 安装 Prometheus 相关组件

```bash
ansible-playbook 本地role.yml文件路径 -e @本地extend.yml文件路径 -e cmd=install -t 需要安装的组件
```

可安装的组件示例：

- **all** ：全部服务
- **server** ：Prometheus Server 相关服务，含：Prometheus Server、Grafana、Alertmanager、Prometheus Alert、Prometheus Webhook Dingtalk、Loki
- **exporter** ：Prometheus Exporter 相关服务，含：Node Exporter、CAdvisor、Mysqld Exporter、Mongodb Exporter、Redis Exporter、Rabbitmq Exporter、Rocketmq Exporter、Elasticsearch Exporter、Zookeeper Exporter、JMX Exporter、Nginx Prometheus Exporter、Blackbox Exporter、Grafana Promtail
- **dockerce** ：Docker 服务
- **node** ：Prometheus Node Exporter
- **docker** ：Prometheus CAdvisor
- **mysql** ：Prometheus Mysqld Exporter
- **mongodb** ：Prometheus Mongodb Exporter
- **redis** ：Prometheus Redis Exporter
- **rabbitmq** ：Prometheus Rabbitmq Exporter
- **rocketmq** ：Prometheus Rocketmq Exporter
- **elasticsearch** ：Prometheus Elasticsearch Exporter
- **zookeeper** ：Prometheus Zookeeper Exporter
- **jmx** ：Prometheus JMX Exporter
- **nginx** ：Nginx Prometheus Exporter
- **blackbox** ：Prometheus Blackbox Exporter
- **promtail** ：Grafana Promtail

安装命令示例：

- 安装全部组件

```bash
ansible-playbook /tmp/ansible/ansible_examples/prometheus/roles/role.yml -e @/tmp/ansible/ansible_examples/prometheus/roles/conf/extend.yml -e cmd=install -t all
```

- 仅安装 server 相关组件

```bash
ansible-playbook /tmp/ansible/ansible_examples/prometheus/roles/role.yml -e @/tmp/ansible/ansible_examples/prometheus/roles/conf/extend.yml -e cmd=install -t server
```

- 仅安装 Prometheus Node Exporter

```bash
ansible-playbook /tmp/ansible/ansible_examples/prometheus/roles/role.yml -e @/tmp/ansible/ansible_examples/prometheus/roles/conf/extend.yml -e cmd=install -t node
```

  

---

附：卸载相关组件

```bash
ansible-playbook 本地role.yml文件路径 -e @本地extend.yml文件路径 -e cmd=remove -t 需要安装的组件
```

示例：

- 卸载全部组件

```bash
ansible-playbook /tmp/ansible/ansible_examples/prometheus/roles/role.yml -e @/tmp/ansible/ansible_examples/prometheus/roles/conf/extend.yml -e cmd=remove -t all
```

- 仅卸载 server 相关组件

```bash
ansible-playbook /tmp/ansible/ansible_examples/prometheus/roles/role.yml -e @/tmp/ansible/ansible_examples/prometheus/roles/conf/extend.yml -e cmd=remove -t server
```

- 仅卸载 Prometheus Node Exporter

```bash
ansible-playbook /tmp/ansible/ansible_examples/prometheus/roles/role.yml -e @/tmp/ansible/ansible_examples/prometheus/roles/conf/extend.yml -e cmd=remove -t node
```

分类: [Prometheus 监控体系](https://hty1024.com/categories/pgal)


