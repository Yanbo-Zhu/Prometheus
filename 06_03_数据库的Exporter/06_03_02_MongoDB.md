
## Mongodb Exporter

1. 目录准备  
    创建目录：

```sh
mkdir -pv /apps/exporter/mongodb-exporter
```

2. 编辑 docker-compose.yml 文件

```sh
vim /apps/exporter/mongodb-exporter/docker-compose.yml
```

```yaml
version: "3"services:  mongodb-exporter:    image: bitnami/mongodb-exporter:0.34.0    container_name: prometheus-mongodb-exporter    hostname: mongodb-exporter    restart: always    ports:      - 9216:9216    command:      - '--mongodb.uri=mongodb://host:port'networks:  default:    external:      name: prometheus
```

参数说明：

- `host` ：MongoDB 主机名称/IP
- `port` ：MongoDB 端口号

3. 配置 docker 网段 prometheus  
    检查是否存在 prometheus 网段：

```sh
docker network list
```

若不存在，则创建：

```sh
docker network create prometheus --subnet 10.21.22.0/24
```

4. 启动 mongodb-exporter 容器

```sh
cd /apps/exporter/mongodb-exporter
```

```sh
docker-compose up -d
```

5. 检查 mongodb-exporter 容器状态、查看 mongodb-exporter 容器日志

```sh
cd /apps/exporter/mongodb-exporter
```

```sh
docker-compose ps
```

```sh
docker-compose logs -f
```

6. 配置 prometheus（ 编辑 prometheus.yml 文件 ）

```yaml
scrape_configs:  - job_name: 'demo-mongodb'    - targets: ['127.0.0.1:9216']
```

7. 重启 prometheus

```sh
cd /apps/prometheus
```

```sh
docker-compose restart
```

8. 检查 Mysqld Exporter 是否正常运行  
    访问 Prometheus WebUI 的 targets 页面，检查 job 状态
    
9. 配置 Grafana  
    登录 Grafana，导入对应的看板即可。  
    看板获取地址：[https://grafana.com/grafana/dashboards/?dataSource=prometheus&search=MongoDB](https://grafana.com/grafana/dashboards/?dataSource=prometheus&search=MongoDB)
    
    **注意： 看板导入后需要修改数据源的ID**
    
    - 数据源查看方式： 在 Grafana 中进入 `数据源详情` 页面，浏览器 URL 的最后一段字符为该数据源的 ID。 如 URL 为 `grafana/datasources/edit/6lbJpCb4z` 时， `6lbJpCb4z` 即为当前数据源的 ID
    - 数据源替换方式： 编辑看板，查看看板的 JSON 数据，替换 `datasource` 中的 `uid`



