

时序数据库是 Promtheus 监控平台的一部分，有着非常高效的时间序列数据存储方法，每个采样数据仅仅占用3.5byte左右空间

在早期有一个单独的项目叫做 TSDB，但是，在2.1.x的某个版本，已经不单独维护这个项目了，直接将这个项目合并到了prometheus的主干上了

 Prometheus将所有数据存储为时间序列；具有相同度量名称以及标签属于同一个指标。

每个时间序列都由度量标准名称和一组键值对（也成为标签）唯一标识。

时间序列格式：
`<metric name>{<label name>=<label value>, ...}`

示例：
api_http_requests_total{method="POST", handler="/messages"}
