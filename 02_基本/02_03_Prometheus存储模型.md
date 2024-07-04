
prometheus使用了G家的LevelDB来做索引(PromSQL重度依赖LevelDB)，对于大量的采样数据有自己的存储层，Prometheus为每个时序数据创建一个本地文件，以1024byte大小的chunk来组织。

时序数据库是 Promtheus 监控平台的一部分，有着非常高效的时间序列数据存储方法，每个采样数据仅仅占用3.5byte左右空间

在早期有一个单独的项目叫做 TSDB，但是，在2.1.x的某个版本，已经不单独维护这个项目了，直接将这个项目合并到了prometheus的主干上了

 Prometheus将所有数据存储为时间序列；具有相同度量名称以及标签属于同一个指标。

每个时间序列都由度量标准名称和一组键值对（也成为标签）唯一标识。

时间序列格式：
`<metric name>{<label name>=<label value>, ...}`

示例：
api_http_requests_total{method="POST", handler="/messages"}


# 1 磁盘文件

Prometheus在storage.local.path指定的路径存储文件，默认为./data。关于chunk编码有三种

- type 0
第一代的编码格式，simple delta encoding

- type 1
目前默认的编码格式，double-delta encoding

- type 2
variable bit-width encoding，facebook的时间序列数据库Beringei采用的编码方式

# 2 内存使用

prometheus在内存里保存了最近使用的chunks，具体chunks的最大个数可以通过storage.local.memory-chunks来设定，默认值为1048576，即1048576个chunk，大小为1G。  
除了采用的数据，prometheus还需要对数据进行各种运算，因此整体内存开销肯定会比配置的local.memory-chunks大小要来的大，因此官方建议要预留3倍的local.memory-chunks的内存大小。

```
As a rule of thumb, you should have at least three times more RAM available than needed by the memory chunks alone
```

可以通过server的metrics去查看prometheus_local_storage_memory_chunks以及process_resident_memory_byte两个指标值。
- prometheus_local_storage_memory_chunks
    - The current number of chunks in memory, excluding cloned chunks 目前内存中暴露的chunks的个数
- process_resident_memory_byte
    - Resident memory size in byte 驻存在内存的数据大小
- prometheus_local_storage_persistence_urgency_score
    - 介于0-1之间，当该值小于等于0.7时，prometheus离开rushed模式。当大于0.8的时候，进入rushed模式
- prometheus_local_storage_rushed_mode
　　`_1表示进入了rushed mode，0表示没有。进入了rushed模式的话，prometheus会利用storage.local.series-sync-strategy以及storage.local.checkpoint-interval的配置加速chunks的持久化。_`

# 3 storage参数

```
docker run -p 9090:9090 \
-v /tmp/prometheus-data:/prometheus-data \
prom/prometheus \
-storage.local.retention 168h0m0s \
-storage.local.max-chunks-to-persist 3024288 \
-storage.local.memory-chunks=50502740 \
-storage.local.num-fingerprint-mutexes=300960
```

## 3.1 storage.local.memory-chunks

设定prometheus内存中保留的chunks的最大个数，默认为1048576，即为1G大小

## 3.2 storage.local.retention

用来配置采用数据存储的时间，168h0m0s即为24*7小时，即1周

## 3.3 storage.local.series-file-shrink-ratio

用来控制序列文件rewrite的时机，默认是在10%的chunks被移除的时候进行rewrite，如果磁盘空间够大，不想频繁rewrite，可以提升该值，比如0.3，即30%的chunks被移除的时候才触发rewrite。

## 3.4 storage.local.max-chunks-to-persist

该参数控制等待写入磁盘的chunks的最大个数，如果超过这个数，Prometheus会限制采样的速率，直到这个数降到指定阈值的95%。建议这个值设定为storage.local.memory-chunks的50%。Prometheus会尽力加速存储速度，以避免限流这种情况的发送。

## 3.5 storage.local.num-fingerprint-mutexes

当prometheus server端在进行checkpoint操作或者处理开销较大的查询的时候，采集指标的操作会有短暂的停顿，这是因为prometheus给时间序列分配的mutexes可能不够用，可以通过这个指标来增大预分配的mutexes，有时候可以设置到上万个。

## 3.6 storage.local.series-sync-strategy

控制写入数据之后，何时同步到磁盘，有'never', 'always', 'adaptive'. 同步操作可以降低因为操作系统崩溃带来数据丢失，但是会降低写入数据的性能。  
默认为adaptive的策略，即不会写完数据就立刻同步磁盘，会利用操作系统的page cache来批量同步。

## 3.7 storage.local.checkpoint-interval

进行checkpoint的时间间隔，即对尚未写入到磁盘的内存chunks执行checkpoint操作。





