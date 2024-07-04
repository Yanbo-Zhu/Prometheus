
# 1 Textfile Collector

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


# 2 Lighthouse

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









