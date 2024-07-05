
influxdb_export 是用来采集influxdb数据的指标的，但是influxdb提供一个专门的一个产品来暴露metrics数据， 也就是说infludb_exporter这个第三方的产品将来会被淘汰了。 不过还是可以使用的，可以参考： [https://github.com/prometheus/influxdb_exporter](https://github.com/prometheus/influxdb_exporter)

infludb官方的工具来获取metrics数据是telegraf, 这个工具相当的强大，内部使用prometheus client插件来暴露数据给prometheus采集， 当然这个工具内部集成了几十种插件用户暴露数据给其他的监控系统。 详细的可以参考官方地址： [https://docs.influxdata.com/telegraf/v1.7/plugins/outputs/#prometheus-client-prometheus-client-https-github-com-influxdata-telegraf-tree-release-1-7-plugins-outputs-prometheus-client](https://docs.influxdata.com/telegraf/v1.7/plugins/outputs/#prometheus-client-prometheus-client-https-github-com-influxdata-telegraf-tree-release-1-7-plugins-outputs-prometheus-client)

这里我们使用的监控系统是prometheus， 只需要关注如下配置即可：  [https://github.com/influxdata/telegraf/tree/release-1.7/plugins/outputs/prometheus_client](https://github.com/influxdata/telegraf/tree/release-1.7/plugins/outputs/prometheus_client)

telegraf的安装配置
```
wget https://dl.influxdata.com/telegraf/releases/telegraf-1.12.2-1.x86_64.rpm
sudo yum localinstall telegraf-1.12.2-1.x86_64.rpm

rpm -ql |grep telegraf
cp /etc/telegraf/telegraf.conf  /etc/telegraf/telegraf.conf.default
# 修改如下部分
 [[outputs.prometheus_client]]
   ## Address to listen on
   listen = ":9273"

systemctl restart telegraf
systemctl status telegraf
systemctl enabletelegraf
```


集成prometheus
```
# prometheus加入如下采集
  - job_name: "influxdb-exporter"
    static_configs:
    - targets: [ "192.168.100.10:9273" ]
```
