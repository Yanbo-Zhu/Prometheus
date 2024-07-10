
# 1 Prometheus Alert 的安装和配置

Docker Compose 方式

创建相关目录
```
mkdir -p /apps/prometheus-alert
```

编辑 docker-compose.yml 文件
```
vim /apps/prometheus-alert/docker-compose.yml

version: "3"
	services:
	  prometheus-alert:
	    image: feiyu563/prometheus-alert:v4.8.2
	    container_name: prometheus-prometheus-alert
	    hostname: prometheus-alert
	    restart: always
	    ports:
	      - 8080:8080
	    volumes:
	      - /etc/localtime:/etc/localtime:ro
	      - /apps/prometheus-alert/conf:/app/conf
	      - /apps/prometheus-alert/db:/app/db
	      - /apps/prometheus-alert/logs:/app/logs
	networks:
	  default:
	    external:
	      name: prometheus
```

3 
编辑 app.conf 文件
Prometheus Alert 的配置文件，模板参考：https://github.com/feiyu563/PrometheusAlert/blob/master/conf/app-example.conf

```
vim /apps/prometheus-alert/conf/app.conf
```

文件内容略，具体内容见上方连接


4
配置 docker 网段 prometheus
检查是否存在 prometheus 网段：
docker network list

若不存在，则创建：
docker network create prometheus --subnet 10.21.22.0/24

5 
启动 Prometheus Alert 容器

```sh
cd /apps/prometheus-alert
docker-compose up -d
```

6 
检查 Prometheus Alert 容器状态、查看 Prometheus Alert 容器日志

```
cd /apps/prometheus-alert
docker-compose ps
docker-compose logs -f
```

7 
配置 Alertmanager
编辑 Alertmanager 的 alertmanager.yml 配置文件，添加 receivers ，参考文档：https://github.com/feiyu563/PrometheusAlert/blob/master/doc/readme/system.md

vim /apps/alertmanager/conf/alertmanager.yml

```
global:
	  resolve_timeout: 5m
	 
	route:
	  group_by: ['instance']
	  group_wait: 10s
	  group_interval: 10s
	  repeat_interval: 1h
	  receiver: 'alert-dd'
	  routes:
	  - receiver: 'alert-dx'
	    continue: true
	    match:
	      severity: emergency
	  - receiver: 'alert-dd'
	    continue: true
	    match_re:
	      severity: warning|critical
	 
	receivers:
	- name: 'alert-dd'
	  webhook_configs:
	  - url: 'http://127.0.0.1:8080/prometheusalert?type=dd&tpl=prometheus-dd-my&ddurl=https://oapi.dingtalk.com/robot/send?access_token=xxx'
	- name: 'alert-dx'
	  webhook_configs:
	  - url: 'http://127.0.0.1:8080/prometheusalert?type=txdx&tpl=prometheus-dx-my&phone=xxx'
	 
	inhibit_rules:
	  - source_match:
	      severity: 'critical'
	    target_match:
	      severity: 'warning'
	    equal: ['instance']
```

注意：
- `type` ： 为通知类型，`dd` 为钉钉通知、`txdx` 为腾讯云短信、 `alydx` 为阿里云短信，其他见官方文档（ [https://feiyu563.gitbook.io/prometheusalert/base-install/base-restful#url-can-shu-jie-shi](https://feiyu563.gitbook.io/prometheusalert/base-install/base-restful#url-can-shu-jie-shi) ）
- `tpl` ： 为通知模板名称
- `access_token` ： 为钉钉机器人的 AccessToken
- `phone` ： 为手机号，多个手机号之间用 `,` 分割



8 
重启 Alertmanager
```
cd /apps/alertmanager
docker-compose restart

```

9 
新增自定义模板
登录 Prometheus Alert ，新增自定义模板文件

钉钉告警模板
模板名称：prometheus-dd-my
模板类型：钉钉
模板用途：Prometheus
模板内容：

```
{{ $var := .externalURL}}{{ range $k,$v:=.alerts }}
	{{if eq $v.status "resolved"}}
	# Prometheus 恢复信息
	## {{$v.labels.alertname}}
	### {{$v.annotations.description}}
	========== 以下为详细信息 ========== <br>
	**告警级别：** {{$v.labels.severity}} <br>
	**告警状态：** {{$v.status}} <br>
	**告警类型：** {{$v.labels.alertname}} <br>
	**告警主题：** {{$v.annotations.summary}} <br>
	**故障资源：** {{$v.labels.instance}} <br>
	**故障时间：** {{TimeFormat $v.startsAt "2006-01-02 15:04:05"}} <br>
	**恢复时间：** {{TimeFormat $v.endsAt "2006-01-02 15:04:05"}} <br>
	**资源标签：** <br>
	- **name：** {{$v.labels.name}} <br>
	{{else}}
	# **Prometheus 告警信息**
	## **{{$v.labels.alertname}}**
	### {{$v.annotations.description}}
	========== 以下为详细信息 ========== <br>
	**告警级别：** {{$v.labels.severity}} <br>
	**告警状态：** {{$v.status}} <br>
	**告警类型：** {{$v.labels.alertname}} <br>
	**告警主题：** {{$v.annotations.summary}} <br>
	**故障资源：** {{$v.labels.instance}} <br>
	**故障时间：** {{TimeFormat $v.startsAt "2006-01-02 15:04:05"}} <br>
	**资源标签：** <br>
	- **name：** {{$v.labels.name}} <br>
	{{end}}
	{{ end }}
	{{ $urimsg:=""}}{{ range $key,$value:=.commonLabels }}{{$urimsg =  print $urimsg $key "%3D%22" $value "%22%2C" }}{{end}}
```

短信告警模板（以 腾讯云 为例，使用其他云供应商时修改 模版类型 即可）
模板名称：prometheus-dx-my
模板类型：腾讯云短信
模板用途：Prometheus
模板内容：

```
{{ range $k,$v:=.alerts }}{{if eq $v.status "resolved"}}
	[Prometheus恢复信息]
	{{$v.labels.alertname}}
	故障描述： {{$v.annotations.description}}
	故障级别： {{$v.labels.severity}}
	故障资源： {{$v.labels.instance}}
	资源标签： 
	- name:   {{$v.labels.name}}
	{{else}}
	[Prometheus告警信息]
	{{$v.labels.alertname}}
	故障描述： {{$v.annotations.description}}
	故障级别： {{$v.labels.severity}}
	故障资源： {{$v.labels.instance}}
	资源标签： 
	- name:   {{$v.labels.name}}
	{{end}}
	{{ end }}
```


注意： 在阿里云新增短信模板时，需要包含 ${code} 变量


10  制造告警，验证告警消息是否发送
示例消息如下：
```
Prometheus 告警信息
主机连接失败
主机 127.0.0.1:9100 连接失败!
========== 以下为详细信息 ==========
告警级别： critical
告警状态： firing
告警类型： 主机连接失败
告警主题： 主机 (127.0.0.1:9100) 连接失败
故障资源： 127.0.0.1:9100
故障时间： 2022-10-12 06:10:34
资源标签：
name： 127.0.0.1
```


```
Prometheus 恢复信息
主机连接失败
主机 127.0.0.1:9100 连接失败!
========== 以下为详细信息 ==========
告警级别： critical
告警状态： resolved
告警类型： 主机连接失败
告警主题： 主机 (172.19.16.17:9100) 连接失败
故障资源： 127.0.0.1:9100
故障时间： 2022-10-12 06:10:34 
恢复时间： 2022-10-12 06:10:49 
资源标签： 
name： 127.0.0.1
```




# 2 Prometheus Webhook Dingtalk 的安装和配置

Docker Compose 方式

1 目录准备
```
mkdir -pv /apps/prometheus-webhook-dingtalk/{conf,template}
```


2 编辑 docker-compose.yml 文件

```
vim /apps/prometheus-webhook-dingtalk/docker-compose.yml
```

```
version: "3"
services:
  prometheus-webhook-dingtalk:
    image: timonwong/prometheus-webhook-dingtalk:v2.1.0
    container_name: prometheus-prometheus-webhook-dingtalk
    hostname: prometheus-webhook-dingtalk
    restart: always
    volumes:
      - /apps/prometheus-webhook-dingtalk/template/template.tmpl:/etc/prometheus-webhook-dingtalk/templates/legacy/template.tmpl
      - /apps/prometheus-webhook-dingtalk/conf/config.yml:/etc/prometheus-webhook-dingtalk/config.yml
    ports:
      - 8060:8060
    environment:
      - config.file=/etc/prometheus-webhook-dingtalk/config.yml
networks:
  default:
    external:
      name: prometheus
```


3 
编辑配置文件（ config.yml ）
参考：https://github.com/timonwong/prometheus-webhook-dingtalk 中的 Configuration 部分

vim /apps/prometheus-webhook-dingtalk/conf/config.yml

```
templates:
  - /etc/prometheus-webhook-dingtalk/templates/legacy/template.tmpl
 
targets:
  webhook1:
    url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxxxxx
    secret: SEC000000000000000000000
  webhook2:
    url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxxxxx
  webhook_legacy:
    url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxxxxx
    message:
      title: '{{ template "legacy.title" . }}'
      text: '{{ template "email.to.message" . }}'
  webhook_mention_all:
    url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxxxxx
    mention:
      all: true
  webhook_mention_users:
    url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxxxxx
    mention:
      mobiles: ['156xxxx8827', '189xxxx8325']
```


4 
编辑消息模板文件（ template.tmpl ）
参考：https://github.com/timonwong/prometheus-webhook-dingtalk/blob/main/template/default.tmpl

vim /apps/prometheus-webhook-dingtalk/template/template.tmpl

```
{{ define "email.to.message" }}
 
{{- if gt (len .Alerts.Firing) 0 -}}
{{- range $index, $alert := .Alerts -}}
 
=========  **监控告警** =========  
 
**告警程序:**     Alertmanager   
**告警类型:**    {{ $alert.Labels.alertname }}   
**告警级别:**    {{ $alert.Labels.severity }} 级   
**告警状态:**    {{ .Status }}   
**故障主机:**    {{ $alert.Labels.instance }} {{ $alert.Labels.device }}   
**告警主题:**    {{ .Annotations.summary }}   
**告警详情:**    {{ $alert.Annotations.message }}{{ $alert.Annotations.description}}   
**主机标签:**    {{ range .Labels.SortedPairs  }}  </br>  [ {{ .Name }}: {{ .Value | markdown | html }} ]   
{{- end }} </br>
 
**故障时间:**    {{ ($alert.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}  
========= = end =  =========  
{{- end }}
{{- end }}
 
{{- if gt (len .Alerts.Resolved) 0 -}}
{{- range $index, $alert := .Alerts -}}
 
========= 告警恢复 =========  
**告警程序:**     Alertmanager   
**告警类型:**    {{ .Labels.alertname }}
**告警级别:**    {{ $alert.Labels.severity }} 级
**告警状态:**    {{   .Status }}
**告警主机:**    {{ .Labels.instance }}
**告警主题:**    {{ $alert.Annotations.summary }}  
**告警详情:**    {{ $alert.Annotations.message }}{{ $alert.Annotations.description}}  
**故障时间:**    {{ ($alert.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}  
**恢复时间:**    {{ ($alert.EndsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}  
 
========= = **end** =  =========
{{- end }}
{{- end }}
{{- end }}
```



5 配置 Alertmanager
编辑 Alertmanager 的 alertmanager.yml 配置文件（需要将 webhook_configs 的 url 修改为 pPrometheus Webhook Dingtalk 的地址）

vim /apps/prometheus-webhook-dingtalk/conf/alertmanager.yml

```
global:
	  resolve_timeout: 5m
	 
	route:
	  group_by: ['alertname']
	  group_wait: 10s
	  group_interval: 10s
	  repeat_interval: 1h
	  receiver: 'dingtalk'
	 
	receivers:
	- name: 'dingtalk'
	  webhook_configs:
	  - url: 'http://127.0.0.1:8060/dingtalk/webhook_legacy/send'
	    send_resolved: true
	 
	inhibit_rules:
	  - source_match:
	      severity: 'critical'
	    target_match:
	      severity: '*'
	    equal: ['instance']
```



6
创建 docker 网段 prometheus

```
检查是否存在 prometheus 网段：
docker network list


若不存在，则创建：
docker network create prometheus --subnet 10.21.22.0/24


启动 Alertmanager 容器
cd /apps/prometheus-webhook-dingtalk
docker-compose up -d


查看 Alertmanager 容器状态、查看  Alertmanager 容器日志
cd /apps/prometheus-webhook-dingtalk
docker-compose ps
docker-compose logs -f

```

9 
验证告警发送是否正常
制造告警，验证钉钉消息是否发送正常，示例如下：
```
=========  监控告警 =========  
告警程序:     Alertmanager
告警类型:    主机连接失败
告警级别:    warning 级
告警状态:    firing
故障主机:    127.0.0.1:9100
告警主题:    主机 (127.0.0.1:9100) 连接失败
告警详情:    主机 127.0.0.1:9100 连接失败!
主机标签:       [name： 127.0.0.1] 
故障时间:    2022-09-28 13:22:02
========= = end =  =========

========= 告警恢复 =========
告警程序:     Alertmanager
告警类型:    主机连接失败
告警级别:    warning 级
告警状态:    resolved
告警主机:    127.0.0.1:9100
告警主题:    主机 (127.0.0.1:9100) 连接失败
告警详情:    主机 127.0.0.1:9100 连接失败!
故障时间:    2022-09-28 13:22:02
恢复时间:    2022-09-28 13:23:02  
========= = end =  =========


```




