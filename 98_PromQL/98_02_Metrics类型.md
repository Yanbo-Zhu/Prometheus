
在上一小节中我们带领读者了解了Prometheus的底层数据模型，在Prometheus的存储实现上所有的监控样本都是以time-series的形式保存在Prometheus内存的TSDB（时序数据库）中，而time-series所对应的监控指标(metric)也是通过labelset进行唯一命名的。

从存储上来讲所有的监控指标metric都是相同的，但是在不同的场景下这些metric又有一些细微的差异。 例如，在Node Exporter返回的样本中指标node_load1反应的是当前系统的负载状态，随着时间的变化这个指标返回的样本数据是在不断变化的。而指标node_cpu所获取到的样本数据却不同，它是一个持续增大的值，因为其反应的是CPU的累积使用时间，从理论上讲只要系统不关机，这个值是会无限变大的。

为了能够帮助用户理解和区分这些不同监控指标之间的差异，Prometheus定义了4种不同的指标类型(metric type)：Counter（计数器）、Gauge（仪表盘）、Histogram（直方图）、Summary（摘要）。

在Exporter返回的样本数据中，其注释中也包含了该样本的类型。例如：

```
# HELP node_cpu Seconds the cpus spent in each mode.
# TYPE node_cpu counter
node_cpu{cpu="cpu0",mode="idle"} 362812.7890625
```


# 1 四种数据类型

- gauge 当前值, 递增的计数器
    - 适合：API 接口请求次数，重试次数。
    - node_memory_MemFree_bytes
- counter 计数器是代表一个累积指标单调递增计数器，其价值只能在重新启动增加或归零。例如，您可以使用计数器来表示已服务请求，已完成任务或错误的数量。
    - 适合：cpu变化，类似波浪线不均匀。
    - http_request_total
- histogram 直方图样本观测（通常之类的东西请求持续时间或响应大小）和计数它们配置的桶中。它还提供所有观察值的总和
    - 适合：将web 一段时间进行分组，根据标签度量名称，统计这段时间这个度量名称有多少条。
    - 适合：某个时间对某个度量值，分组，一段时间http相应大小，请求耗时的时间。
```
# http所有接口 总的95分位值
# sum/count 可以算平均值
prometheus_http_request_duration_seconds_sum/ prometheus_http_request_duration_seconds_count

# histogram_quantile(0.95, sum(rate(prometheus_http_request_duration_seconds_bucket[5m])) by (le,handler))

histogram_quantile(0.95, sum(rate(prometheus_http_request_duration_seconds_bucket[1m])) by (le))

# range_query接口的95分位值
histogram_quantile(0.95, sum(rate(prometheus_http_request_duration_seconds_bucket{handler="/api/v1/query_range"}[5m])) by (le))


```

- summary  摘要会采样观察值（通常是请求持续时间和响应大小之类的东西）。尽管它还提供了观测值的总数和所有观测值的总和，但它可以计算滑动时间窗口内的可配置分位数。

```
# gc耗时

# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0.000135743
go_gc_duration_seconds{quantile="0.25"} 0.000872805
go_gc_duration_seconds{quantile="0.5"} 0.000965516
go_gc_duration_seconds{quantile="0.75"} 0.001055636
go_gc_duration_seconds{quantile="1"} 0.006464756

# summary 平均值
go_gc_duration_seconds_sum /go_gc_duration_seconds_count

```


# 2 哪些指标可以作为Metric，可以根据不同系统做分类：

Online-serving systems在线服务器，比如SRS或API服务器，需要立刻对请求做响应的服务器。关键指标是请求数目、错误数目、请求耗时、服务器并发。应该在客户端和服务器同时采集，当两边的数据不同时会有助于分析问题；当然如果并发太高，就只能靠自己的统计系统了。一般是在请求结束时采集数据，比较方便采集错误和耗时。

Offline processing离线处理。离线处理就不会在线等响应，离线处理一般是批处理完成的，而且会分成很多阶段处理。关键指标是每个阶段的输入输出，多少在处理中，上次处理完的时间；对于批处理，还需要跟踪分支。当任务挂起时，知道最后任务的完成时间很重要，更好的是任务的心跳。

Batch jobs批处理。批处理和离线处理有时候有点混淆，因为离线处理有时候用批处理实现。批处理的关键特点是非连续性，所以很难抓取有效的指标。关键指标是最后成功的时间和结果(错误码)，采集每个主要阶段的耗时也很重要。应该推送给PushGateway。任务级别的统计数据也很重要，比如处理的记录数。对于长时间运行的任务，基于pull的监控也很重要，可以度量系统的资源变化。对于频繁运行的批处理任务，比如每15分钟执行一次，转成离线处理服务会更好。



# 3 除了这几种类型的系统，还有一些系统的部分可以采集Metric：

Libraries 库的指标采集，应该不需要用户配置。
Logging 日志的指标，应该有个总的日志数，对于某个特别的日志应该看它的频率和耗时，对于函数的某些分支也可以采集，统计不同级别的日志数量也挺有用的。
Failures 错误，和日志类似应该采集总数。以及总请求数，这样比较容易计算错误率。
Threadpools 线程池，关键指标是排队的请求数，活跃的线程数，总线程数，处理的任务和耗时，队列的等待耗时也很有用。
Cache 缓存，关键指标是总查询数，命中数目，延迟，错误数。
Collectors Prometheus的Collector，采集的耗时和错误次数。这里建议用gauge而不是histogram，另外一个用gauge的例子是批处理的耗时，这是因为它们度量的是每次的推送或抓取，而不是分散的多个耗时。



# 4 到底该用哪种类型的指标：

Counter vs Gauge：能变小的是Gauge，比如占用内存大小。
Counter是递增的，直到进程重启会被Reset，比如处理的总请求数、总错误数、发送的总字节数。一般Counter很少能直接使用，是需要函数处理比如rate或做其他处理。
Gauge的值可以被设置，可以变大变小，一般是状态的快照，比如处理中的请求数，可用的内存大小，温度等，对于gauge永远不要用rate函数。
Summary和Histogram比较难，用得也比较少，大概是统计分布，比如耗时的分布，暂时没仔细看。


# 5 Counter：只增不减的计数器

Counter类型的指标其工作方式和计数器一样，只增不减（除非系统发生重置）。常见的监控指标，如http_requests_total，node_cpu都是Counter类型的监控指标。 一般在定义Counter类型指标的名称时推荐使用_total作为后缀。

Counter是一个简单但有强大的工具，例如我们可以在应用程序中记录某些事件发生的次数，通过以时序的形式存储这些数据，我们可以轻松的了解该事件产生速率的变化。 PromQL内置的聚合操作和函数可以让用户对这些数据进行进一步的分析：

例如，通过rate()函数获取HTTP请求量的增长率：
```
rate(http_requests_total[5m])
```


查询当前系统中，访问量前10的HTTP地址：
```
topk(10, http_requests_total)
```



# 6 Gauge：可增可减的仪表盘

与Counter不同，Gauge类型的指标侧重于反应系统的当前状态。因此这类指标的样本数据可增可减。
常见指标如：node_memory_MemFree（主机当前空闲的内容大小）、node_memory_MemAvailable（可用内存大小）都是Gauge类型的监控指标。


通过Gauge指标，用户可以直接查看系统的当前状态：
```
node_memory_MemFree
```

对于Gauge类型的监控指标，通过PromQL内置函数delta()可以获取样本在一段时间返回内的变化情况。例如，计算CPU温度在两个小时内的差异：
```
delta(cpu_temp_celsius{host="zeus"}[2h])
```

还可以使用deriv()计算样本的线性回归模型，甚至是直接使用predict_linear()对数据的变化趋势进行预测。例如，预测系统磁盘空间在4个小时之后的剩余情况：
```
predict_linear(node_filesystem_free{job="node"}[1h], 4 * 3600)
```



# 7 使用Histogram和Summary分析数据分布情况

除了Counter和Gauge类型的监控指标以外，Prometheus还定义了Histogram和Summary的指标类型。Histogram和Summary主用用于统计和分析样本的分布情况。

在大多数情况下人们都倾向于使用某些量化指标的平均值，例如CPU的平均使用率、页面的平均响应时间。这种方式的问题很明显，以系统API调用的平均响应时间为例：如果大多数API请求都维持在100ms的响应时间范围内，而个别请求的响应时间需要5s，那么就会导致某些WEB页面的响应时间落到中位数的情况，而这种现象被称为长尾问题。

为了区分是平均的慢还是长尾的慢，最简单的方式就是按照请求延迟的范围进行分组。例如，统计延迟在0~10ms之间的请求数有多少而10~20ms之间的请求数又有多少。通过这种方式可以快速分析系统慢的原因。Histogram和Summary都是为了能够解决这样问题的存在，通过Histogram和Summary类型的监控指标，我们可以快速了解监控样本的分布情况。

---
Summary 类型的 metric

例如，指标prometheus_tsdb_wal_fsync_duration_seconds的指标类型为Summary。 它记录了Prometheus Server中wal_fsync处理的处理时间，通过访问Prometheus Server的/metrics地址，可以获取到以下监控样本数据：

```
# 1 HELP prometheus_tsdb_wal_fsync_duration_seconds Duration of WAL fsync.
# 2 TYPE prometheus_tsdb_wal_fsync_duration_seconds summary
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.5"} 0.012352463
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.9"} 0.014458005
prometheus_tsdb_wal_fsync_duration_seconds{quantile="0.99"} 0.017316173
prometheus_tsdb_wal_fsync_duration_seconds_sum 2.888716127000002
prometheus_tsdb_wal_fsync_duration_seconds_count 216
```

从上面的样本中可以得知当前Prometheus Server进行wal_fsync操作的总次数为216次，耗时2.888716127000002s。其中中位数（quantile=0.5）的耗时为0.012352463，9分位数（quantile=0.9）的耗时为0.014458005s。

----
Histogram类型的metric 

在Prometheus Server自身返回的样本数据中，我们还能找到类型为Histogram的监控指标prometheus_tsdb_compaction_chunk_range_bucket。

```
# 3 HELP prometheus_tsdb_compaction_chunk_range Final time range of chunks on their first compaction
# 4 TYPE prometheus_tsdb_compaction_chunk_range histogram
prometheus_tsdb_compaction_chunk_range_bucket{le="100"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="400"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="1600"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="6400"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="25600"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="102400"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="409600"} 0
prometheus_tsdb_compaction_chunk_range_bucket{le="1.6384e+06"} 260
prometheus_tsdb_compaction_chunk_range_bucket{le="6.5536e+06"} 780
prometheus_tsdb_compaction_chunk_range_bucket{le="2.62144e+07"} 780
prometheus_tsdb_compaction_chunk_range_bucket{le="+Inf"} 780
prometheus_tsdb_compaction_chunk_range_sum 1.1540798e+09
prometheus_tsdb_compaction_chunk_range_count 780
```

与Summary类型的指标相似之处在于Histogram类型的样本同样会反应当前指标的记录的总数(以_count作为后缀)以及其值的总量（以_sum作为后缀）。不同在于Histogram指标直接反应了在不同区间内样本的个数，区间通过标签len进行定义。

同时对于Histogram的指标，我们还可以通过histogram_quantile()函数计算出其值的分位数。==不同在于Histogram通过histogram_quantile函数是在服务器端计算的分位数。 而Sumamry的分位数则是直接在客户端计算完成。因此对于分位数的计算而言，Summary在通过PromQL进行查询时有更好的性能表现，而Histogram则会消耗更多的资源。反之对于客户端而言Histogram消耗的资源更少。在选择这两种方式时用户应该按照自己的实际场景进行选择。==





