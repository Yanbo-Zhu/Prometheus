
默认情况下prometheus会将采集的数据防止到本机的data目录的， 存储数据的大小受限和扩展不便，这是使用influxdb作为后端的数据库来存储数据。


# 1 influxdb安装配置

influxdb的官方文档地址为： https://docs.influxdata.com/influxdb/v1.7/introduction/downloading/ 根据不同系统进行下载，这里使用官方提供的rpm进行安装。


```
# 下载rpmwget https://dl.influxdata.com/influxdb/releases/influxdb-1.7.8.x86_64.rpm# 本地安装rpm
sudo yum localinstall influxdb-1.7.8.x86_64.rpm# 查看安装的文件
[root@node00 influxdb]# rpm -ql influxdb
/etc/influxdb/influxdb.conf
/etc/logrotate.d/influxdb
/usr/bin/influx
/usr/bin/influx_inspect
/usr/bin/influx_stress
/usr/bin/influx_tsm
/usr/bin/influxd
/usr/lib/influxdb/scripts/influxdb.service
/usr/lib/influxdb/scripts/init.sh
/usr/share/man/man1/influx.1.gz
/usr/share/man/man1/influx_inspect.1.gz
/usr/share/man/man1/influx_stress.1.gz
/usr/share/man/man1/influx_tsm.1.gz
/usr/share/man/man1/influxd-backup.1.gz
/usr/share/man/man1/influxd-config.1.gz
/usr/share/man/man1/influxd-restore.1.gz
/usr/share/man/man1/influxd-run.1.gz
/usr/share/man/man1/influxd-version.1.gz
/usr/share/man/man1/influxd.1.gz
/var/lib/influxdb
/var/log/influxdb# 备份默认的默认的配置文件，这里可以对influxdb的数据存放位置做些设置[root@node00 influxdb]# cp /etc/influxdb/influxdb.conf  /etc/influxdb/influxdb.conf.default# 启动
[root@node00 influxdb]# systemctl restart influxdb# 查看状态
[root@node00 influxdb]# systemctl status influxdb
# 客户端登陆测试， 创建一个prometheus的database供后续的prometheus使用。
[root@node00 influxdb]# influx
Connected to http://localhost:8086 version 1.7.8
InfluxDB shell version: 1.7.8
> show databases;
name: databases
name
----
_internal
> create database prometheus;
> show databases;
name: databases
name
----
_internal
prometheus
> exit
```



# 2 配置prometheus集成infludb


官方的帮助文档在这里： https://docs.influxdata.com/influxdb/v1.7/supported_protocols/prometheus/

注意： 如果influxdb配置有密码， 请参考上面的官方文档地址进行配置。
```
[root@node00 prometheus]# pwd
/usr/local/prometheus/prometheus
cp prometheus.yml  prometheus.yml.default
vim prometheus.yml
# 添加如下几行
remote_write:
  - url: "http://localhost:8086/api/v1/prom/write?db=prometheus"

remote_read:
  - url: "http://localhost:8086/api/v1/prom/read?db=prometheus"

systemctl restart prometheus 
systemctl status prometheus
```



# 3 测试数据是否存储到influxdb中


```sh
[root@node00 prometheus]# influx
Connected to http://localhost:8086 version 1.7.8
InfluxDB shell version: 1.7.8
> show databases;
name: databases
name
----
_internal
prometheus
> use prometheus
Using database prometheus
> show measures;
ERR: error parsing query: found measures, expected CONTINUOUS, DATABASES, DIAGNOSTICS, FIELD, GRANTS, MEASUREMENT, MEASUREMENTS, QUERIES, RETENTION, SERIES, SHARD, SHARDS, STATS, SUBSCRIPTIONS, TAG, USERS at line 1, char 6
> show MEASUREMENTS;
name: measurements
name
----
go_gc_duration_seconds
go_gc_duration_seconds_count
go_gc_duration_seconds_sum
go_goroutines
go_info
go_memstats_alloc_bytes
# 后面还是有很多，这里不粘贴了。

# 做个简单查询
> select * from prometheus_http_requests_total limit 10 ; 
name: prometheus_http_requests_total
time                __name__                       code handler  instance       job        value
----                --------                       ---- -------  --------       ---        -----
1568975686217000000 prometheus_http_requests_total 200  /metrics localhost:9090 prometheus 1
1568975701216000000 prometheus_http_requests_total 200  /metrics localhost:9090 prometheus 2
1568975716218000000 prometheus_http_requests_total 200  /metrics localhost:9090 prometheus 3
1568975731217000000 prometheus_http_requests_total 200  /metrics localhost:9090 prometheus 4
1568975746216000000 prometheus_http_requests_total 200  /metrics localhost:9090 prometheus 5
1568975761217000000 prometheus_http_requests_total 200  /metrics localhost:9090 prometheus 6
1568975776217000000 prometheus_http_requests_total 200  /metrics localhost:9090 prometheus 7
1568975791217000000 prometheus_http_requests_total 200  /metrics localhost:9090 prometheus 8
1568975806217000000 prometheus_http_requests_total 200  /metrics localhost:9090 prometheus 9
1568975821216000000 prometheus_http_requests_total 200  /metrics localhost:9090 prometheus 10
```










