
https://www.cnblogs.com/JetpropelledSnake/p/11425626.html


# 1 **日志类：**

-log.level 可选值 [debug, info, warn, error, fatal]  例：-log.level "info"

-log.format  可选输出到syslog或者控制台  例：-log.format "logger:syslog?appname=prom&local=7"

# 2 **查询类：**

      -query.max-concurrency 20  最大支持的并发查询量

　  -query.staleness-delta 5m0s    Staleness delta allowance during expression evaluations.

      -query.timeout 2m0s   查询超时时间，2分钟。超过自动被kill掉。

# 3 **存储类：**

　　-storage.local.checkpoint-dirty-series-limit 5000   崩溃恢复时候，只恢复5000个时序数据，这样减少了prometheus的恢复时间。如果是SSD盘，可以适当增大这个值。

　　-storage.local.checkpoint-interval 5m0s   5分钟执行一次落盘，将in-memory metrics and chunks持久化到磁盘。

　　-storage.local.chunk-encoding-version 1     chunks的编码格式 ，默认是1

　　-storage.local.engine "persisted"    开启持久化

　　-storage.local.index-cache-size.label-name-to-label-values 10485760     存放prometheus里面定义的 label名称的 index cache大小，默认10MB

　　-storage.local.path "/bdata/data/nowdb2"

　　-storage.local.retention 8760h0m0s    保存1年的数据

　　-storage.local.series-file-shrink-ratio 0.3    表示 30%的chunks被移除的时候才触发rewrite

　　-storage.local.num-fingerprint-mutexes 4096 当prometheus server端在进行checkpoint操作或者处理开销较大的查询的时候，采集指标的操作会有短暂的停顿，这是因为prometheus给时间序列分配的mutexes可能不够用，可以通过这个指标来增大预分配的mutexes，有时候可以设置到上万个。

　　-storage.local.series-sync-strategy "adaptive"

　　-storage.local.target-heap-size 2147483648     # prometheus独占的内存空间，默认2GB的内存空间，建议不要超过3GB

# 4 **Web配置：**

　　-web.listen-address ":9090"

　　-web.max-connections 512

　　-web.read-timeout 30s


1 
prometheus.yml文件配置修改后，要想重新加载，必须在启动的时候添加参数：
--web.enable-lifecycle  

比如：
nohup ./prometheus --web.enable-lifecycle --config.file=prometheus.yml &



# 5 **目前在用的启动参数：**

![复制代码](https://assets.cnblogs.com/images/copycode.gif)

nohup ./prometheus   
-log.level "info"   
-log.format "logger:syslog?appname=prom&local=7" info   
-storage.local.checkpoint-dirty-series-limit 5000   
-storage.local.checkpoint-interval 5m0s   
-storage.local.chunk-encoding-version 1   
-storage.local.engine "persisted"   
-storage.local.index-cache-size.label-name-to-label-values 10485760   
-storage.local.path "/bdata/data/nowdb2" -storage.local.retention 8760h0m0s   
-storage.local.series-file-shrink-ratio 0.3   
-storage.local.series-sync-strategy "adaptive"   
-storage.local.target-heap-size 2147483648 & 

![复制代码](https://assets.cnblogs.com/images/copycode.gif)

# 6 **关闭进程：**

  kill -SIGTERM $(pid of prometheus)

# 7 **补充：** **./prometheus -h的结果：**

`usage: prometheus [<args>]`

   -version false

      Print version information.

   -config.file "prometheus.yml"

      Prometheus configuration file name.

# 8 ALERTMANAGER

   -alertmanager.notification-queue-capacity 10000
           The capacity of the queue for pending alert manager notifications.

   -alertmanager.timeout 10s
           Alert manager HTTP API timeout.

   -alertmanager.url 
          Comma-separated list of Alertmanager URLs to send notifications to.


# 9 **LOG**

   -log.format "\"logger:stderr\""
      Set the log target and format. Example: 
      "logger:syslog?appname=bob&local=7" or "logger:stdout?json=true"

   -log.level "\"info\""
      Only log messages with the given severity or above. Valid levels: 
     ` [debug, info, warn, error, fatal]`

# 10 **QUERY**

   -query.max-concurrency 20    最大支持的并发查询量

      Maximum number of queries executed concurrently.

   -query.staleness-delta 5m0s

      Staleness delta allowance during expression evaluations.

   -query.timeout 2m0s   查询超时时间，2分钟。超过自动被kill掉。

      Maximum time a query may take before being aborted.

# 11 **STORAGE** 

   -storage.local.checkpoint-dirty-series-limit 5000   崩溃恢复时候，只恢复5000个时序数据，这样减少了prometheus的恢复时间。如果是SSD盘，可以适当增大这个值。

      If approx. that many time series are in a state that would require 

      a recovery operation after a crash, a checkpoint is triggered, even if 

      the checkpoint interval hasn't passed yet. A recovery operation requires 

      a disk seek. The default limit intends to keep the recovery time below 

      1min even on spinning disks. With SSD, recovery is much faster, so you 

      might want to increase this value in that case to avoid overly frequent 

      checkpoints. Also note that a checkpoint is never triggered before at 

      least as much time has passed as the last checkpoint took.

   -storage.local.checkpoint-interval 5m0s   5分钟执行一次落盘，将in-memory metrics and chunks持久化到磁盘。

      The time to wait between checkpoints of in-memory metrics and 

      chunks not yet persisted to series files. Note that a checkpoint is never 

      triggered before at least as much time has passed as the last checkpoint 

      took.

   -storage.local.chunk-encoding-version 1  chunks的编码格式 ，默认是1

      Which chunk encoding version to use for newly created chunks. 

      Currently supported is 0 (delta encoding), 1 (double-delta encoding), and 

      2 (double-delta encoding with variable bit-width).

   -storage.local.dirty=false   是否强制开启crash recovery功能。默认 -storage.local.dirty=false的。

      如果您怀疑数据库中的损坏引起的问题，可设置启动的时候 -storage.local.dirty=true强制执行crash recovery

  If set, the local storage layer will perform crash recovery even if 

      the last shutdown appears to be clean.

   -storage.local.engine "persisted"

      Local storage engine. Supported values are: 'persisted' (full local 

      storage with on-disk persistence) and 'none' (no local storage).

   -storage.local.index-cache-size.fingerprint-to-metric 10485760

      The size in bytes for the fingerprint to metric index cache.

   -storage.local.index-cache-size.fingerprint-to-timerange 5242880

      The size in bytes for the metric time range index cache.

上面2个参数的作用： Increase the size if you have a large number of archived time series, i.e. series that have not received samples in a while but are still not old enough to be purged completely.   

   -storage.local.index-cache-size.label-name-to-label-values 10485760     存放prometheus里面定义的 label名称的 index cache大小，默认10MB

      The size in bytes for the label name to label values index cache.

   -storage.local.index-cache-size.label-pair-to-fingerprints 20971520  # 

      The size in bytes for the label pair to fingerprints index cache. Increase the size if a large number of time series share the same label pair or name.

   -storage.local.max-chunks-to-persist 0   废弃的参数

      Deprecated. This flag has no effect anymore.

   -storage.local.memory-chunks 0  废弃的参数 设定prometheus内存中保留的chunks的最大个数

      Deprecated. If set, -storage.local.target-heap-size will be set to 

      this value times 3072.

   -storage.local.num-fingerprint-mutexes 4096

      The number of mutexes used for fingerprint locking.

当prometheus server端在进行checkpoint操作或者处理开销较大的查询的时候，采集指标的操作会有短暂的停顿，这是因为prometheus给时间序列分配的mutexes可能不够用，可以通过这个指标来增大预分配的mutexes，有时候可以设置到上万个。

   -storage.local.path "data"

      Base path for metrics storage.

   -storage.local.pedantic-checks false   默认false 如果设置true，崩溃恢复时候会检查每一个序列文件

      If set, a crash recovery will perform checks on each series file. 

      This might take a very long time.

   -storage.local.retention 360h0m0s   历史数据存储多久，默认15天。

      How long to retain samples in the local storage.

   -storage.local.series-file-shrink-ratio 0.1

      A series file is only truncated (to delete samples that have 

      exceeded the retention period) if it shrinks by at least the provided 

      ratio. This saves I/O operations while causing only a limited storage 

      space overhead. If 0 or smaller, truncation will be performed even for a 

      single dropped chunk, while 1 or larger will effectively prevent any 

      truncation.

用来控制序列文件rewrite的时机，默认是在10%的chunks被移除的时候进行rewrite，如果磁盘空间够大，不想频繁rewrite，可以提升该值，比如0.3，即30%的chunks被移除的时候才触发rewrite。

   -storage.local.series-sync-strategy "adaptive"

      When to sync series files after modification. Possible values: 

      'never', 'always', 'adaptive'. Sync'ing slows down storage performance 

      but reduces the risk of data loss in case of an OS crash. With the 

      'adaptive' strategy, series files are sync'd for as long as the storage 

      is not too much behind on chunk persistence.

控制写入数据之后，何时同步到磁盘，有'never', 'always', 'adaptive'. 同步操作可以降低因为操作系统崩溃带来数据丢失，但是会降低写入数据的性能。

默认为adaptive的策略，即不会写完数据就立刻同步磁盘，会利用操作系统的page cache来批量同步。

   -storage.local.target-heap-size 2147483648     # prometheus独占的内存空间，默认2GB的内存空间，建议不要超过3GB

      The metrics storage attempts to limit its own memory usage such 

      that the total heap size approaches this value. Note that this is not a 

      hard limit. Actual heap size might be temporarily or permanently higher 

      for a variety of reasons. The default value is a relatively safe setting 

      to not use more than 3 GiB physical memory.

   -storage.remote.graphite-address 

      WARNING: THIS FLAG IS UNUSED! Built-in support for InfluxDB, 

      Graphite, and OpenTSDB has been removed. Use Prometheus's generic remote 

      write feature for building remote storage integrations. See 

      https://prometheus.io/docs/operating/configuration/#<remote_write>

   -storage.remote.graphite-prefix 

      WARNING: THIS FLAG IS UNUSED! Built-in support for InfluxDB, 

      Graphite, and OpenTSDB has been removed. Use Prometheus's generic remote 

      write feature for building remote storage integrations. See 

      https://prometheus.io/docs/operating/configuration/#<remote_write>

   -storage.remote.graphite-transport 

      WARNING: THIS FLAG IS UNUSED! Built-in support for InfluxDB, 

      Graphite, and OpenTSDB has been removed. Use Prometheus's generic remote 

      write feature for building remote storage integrations. See 

      https://prometheus.io/docs/operating/configuration/#<remote_write>

   -storage.remote.influxdb-url 

      WARNING: THIS FLAG IS UNUSED! Built-in support for InfluxDB, 

      Graphite, and OpenTSDB has been removed. Use Prometheus's generic remote 

      write feature for building remote storage integrations. See 

      https://prometheus.io/docs/operating/configuration/#<remote_write>

   -storage.remote.influxdb.database 

      WARNING: THIS FLAG IS UNUSED! Built-in support for InfluxDB, 

      Graphite, and OpenTSDB has been removed. Use Prometheus's generic remote 

      write feature for building remote storage integrations. See 

      https://prometheus.io/docs/operating/configuration/#<remote_write>

   -storage.remote.influxdb.retention-policy 

      WARNING: THIS FLAG IS UNUSED! Built-in support for InfluxDB, 

      Graphite, and OpenTSDB has been removed. Use Prometheus's generic remote 

      write feature for building remote storage integrations. See 

      https://prometheus.io/docs/operating/configuration/#<remote_write>

   -storage.remote.influxdb.username 

      WARNING: THIS FLAG IS UNUSED! Built-in support for InfluxDB, 

      Graphite, and OpenTSDB has been removed. Use Prometheus's generic remote 

      write feature for building remote storage integrations. See 

      https://prometheus.io/docs/operating/configuration/#<remote_write>

   -storage.remote.opentsdb-url 

      WARNING: THIS FLAG IS UNUSED! Built-in support for InfluxDB, 

      Graphite, and OpenTSDB has been removed. Use Prometheus's generic remote 

      write feature for building remote storage integrations. See 

      https://prometheus.io/docs/operating/configuration/#<remote_write>

   -storage.remote.timeout 

      WARNING: THIS FLAG IS UNUSED! Built-in support for InfluxDB, 

      Graphite, and OpenTSDB has been removed. Use Prometheus's generic remote 

      write feature for building remote storage integrations. See 

      https://prometheus.io/docs/operating/configuration/#<remote_write>

 == WEB ==

   -web.console.libraries "console_libraries"

      Path to the console library directory.

   -web.console.templates "consoles"

      Path to the console template directory, available at /consoles.

   -web.enable-remote-shutdown false

      Enable remote service shutdown.

   -web.external-url 

      The URL under which Prometheus is externally reachable (for 

      example, if Prometheus is served via a reverse proxy). Used for 

      generating relative and absolute links back to Prometheus itself. If the 

      URL has a path portion, it will be used to prefix all HTTP endpoints 

      served by Prometheus. If omitted, relevant URL components will be derived 

      automatically.

   -web.listen-address ":9090"

      Address to listen on for the web interface, API, and telemetry.

   -web.max-connections 512

      Maximum number of simultaneous connections.

   -web.read-timeout 30s

      Maximum duration before timing out read of the request, and closing 

      idle connections.

   -web.route-prefix 

      Prefix for the internal routes of web endpoints. Defaults to path 

      of -web.external-url.

   -web.telemetry-path "/metrics"

      Path under which to expose metrics.

   -web.user-assets 

      Path to static asset directory, available at /user.








