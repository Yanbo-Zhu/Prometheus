
# 1 #

![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190921111241432-425608334.png)


![](https://img2018.cnblogs.com/blog/429277/201909/429277-20190921111334050-1689683596.png)



# 2 kubectl logs

Node Exporter has numerous collectors designed to gather OS and hardware metrics from various sources on a node. If you check the log output from a Node Exporter Pod using `kubectl logs` you can see the collectors that are active:

```shell
$kubectl logs node-exporter-c8cwp --namespace prometheus
time="2019-07-04T15:47:47Z" level=info msg="Enabled collectors:" source="node_exporter.go:97"
time="2019-07-04T15:47:47Z" level=info msg=" - arp" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - bcache" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - bonding" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - conntrack" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - cpu" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - cpufreq" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - diskstats" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - edac" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - entropy" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - filefd" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - filesystem" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - hwmon" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - infiniband" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - ipvs" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - loadavg" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - mdadm" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - meminfo" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - netclass" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - netdev" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - netstat" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - nfs" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - nfsd" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - pressure" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - sockstat" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - stat" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - textfile" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - time" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - timex" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - uname" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - vmstat" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - xfs" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - zfs" source="node_exporter.go:104"
```


If you refer back to the Node Exporter [documenation](https://github.com/prometheus/node_exporter#collectors) you can see the method that each of these collectors uses to acquire metrics. For example, the **arp** collector exposes the metrics available in **/proc/net/arp** on Linux.

In Prometheus, you will see that the majority of metrics exposed by the Node Exporter are prefixed with **node__**. For example, the arp collector described above exposes a metric called **node_arp_entries** that contains the number of ARP entries in the ARP table for each network interface on a node.



# 3 Textfile Collector

Node Exporter可以采集文件，比如/etc/node-exporter/node-meta.prom，或者指定路径：
    mkdir node0 && echo 'machine_role{role="apache"} 1' > node0/roles.prom


启动时指定采集这个目录：
```
docker run --rm -it -p 9100:9100 -v $(pwd):/data -w /data \
    prom/node-exporter --collector.textfile.directory node0
```


可以看到这个采集的数据：

machine_role
machine_role{instance="9100", job="node0", role="apache"} 1


# 4 Lighthouse

在LightHouse上运行Prometheus：

docker run --rm --add-host=mgmt.srs.local:10.0.24.2 \
  -v `pwd`/prometheus.yml:/etc/prometheus/prometheus.yml \
  -p 9090:9090 ccr.ccs.tencentyun.com/ossrs/prometheus


运行Node Exporter：

docker run --rm --net=host --pid=host -v "/:/host:ro,rslave" \
  ccr.ccs.tencentyun.com/ossrs/node-exporter --path.rootfs=/host


注意，若启动Prometheus时，指定了data目录，需要使用root启动docker，即--user root，否则访问失败，因为它使用特定的用户运行：
```

docker run --user root --rm --add-host=mgmt.srs.local:10.0.24.2 \
  -p 9090:9090/tcp -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v $(pwd)containers/data/prometheus:/prometheus \
  ccr.ccs.tencentyun.com/ossrs/prometheus
```









