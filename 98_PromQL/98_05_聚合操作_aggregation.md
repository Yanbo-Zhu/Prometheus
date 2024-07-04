
Prometheus还提供了下列内置的聚合操作符，这些操作符作用域瞬时向量。可以将瞬时表达式返回的样本数据进行聚合，形成一个新的时间序列。
- `sum` (求和)
- `min` (最小值)
- `max` (最大值)
- `avg` (平均值)
- `stddev` (标准差)
- `stdvar` (标准方差)
- `count` (计数)
- `count_values` (对value进行计数)
- `bottomk` (后n条时序)
- `topk` (前n条时序)
- `quantile` (分位数)

使用聚合操作的语法如下：
```
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```

其中只有`count_values`, `quantile`, `topk`, `bottomk`支持参数(parameter)。


min_over time()
max_over time ()
avg_over time ()
sum_over time ()
stddev_over time ()
stdvar_over time()
quantile_over_time()
last_over_time()
count_over_time()



**![](https://img2022.cnblogs.com/blog/1354564/202205/1354564-20220522222836931-648255156.png)



# 1 使用聚合操作

一般来说，如果描述样本特征的标签(label)在并非唯一的情况下，通过PromQL查询数据，会返回多条满足这些特征维度的时间序列。而PromQL提供的聚合操作可以用来对这些时间序列进行处理，形成一条新的时间序列：

```
# 查询系统所有http请求的总量
sum(http_request_total)

# 按照mode计算主机CPU的平均使用时间
avg(node_cpu) by (mode)

# 按照主机查询各个主机的CPU使用率
sum(sum(irate(node_cpu{mode!='idle'}[5m]))  / sum(irate(node_cpu[5m]))) by (instance)
```


Aggregatable operators聚合操作，针对Instant vector，按照某些标签聚合，比如cpu有很多标签：
```
node_cpu_seconds_total{cpu=~"0|1",job="node0"}
node_cpu_seconds_total{cpu="0", instance="host.docker.internal:9100", job="node0", mode="idle"} 250784.34
node_cpu_seconds_total{cpu="0", instance="host.docker.internal:9100", job="node0", mode="nice"} 0
node_cpu_seconds_total{cpu="0", instance="host.docker.internal:9100", job="node0", mode="system"} 84181.76
node_cpu_seconds_total{cpu="0", instance="host.docker.internal:9100", job="node0", mode="user"} 49026.15
node_cpu_seconds_total{cpu="1", instance="host.docker.internal:9100", job="node0", mode="idle"} 361174.34
node_cpu_seconds_total{cpu="1", instance="host.docker.internal:9100", job="node0", mode="nice"} 0
node_cpu_seconds_total{cpu="1", instance="host.docker.internal:9100", job="node0", mode="system"} 13072.18
node_cpu_seconds_total{cpu="1", instance="host.docker.internal:9100", job="node0", mode="user" 8485.57
```


我们可以按照mode聚合，这样就可以看到整体不同mode的数据：
```
sum by(mode) (node_cpu_seconds_total{cpu=~"0|1",job="node0"})
{mode="idle"} 612159.87
{mode="nice"} 0
{mode="system"}97277.25
{mode="user"} 57527.03
```



# 2 without

without用于从计算结果中移除列举的标签，而保留其它标签。by则正好相反，结果向量中只保留列出的标签，其余标签则移除。通过without和by可以按照样本的问题对数据进行聚合。

例如：
```
sum(http_requests_total) without (instance)
```

等价于
```
sum(http_requests_total) by (code,handler,job,method)
```

# 3 sum

如果只需要计算整个应用的HTTP请求总量，可以直接使用表达式：
```
sum(http_requests_total)
```

# 4 count_values

count_values用于时间序列中每一个样本值出现的次数。count_values会为每一个唯一的样本值输出一个时间序列，并且每一个时间序列包含一个额外的标签。

例如：
```
count_values("count", http_requests_total)
```

# 5 topk和bottomk


topk和bottomk则用于对样本值进行排序，返回当前样本值前n位，或者后n位的时间序列。

获取HTTP请求数前5位的时序样本数据，可以使用表达式：
```
topk(5, http_requests_total)
```



# 6 quantile 

quantile用于计算当前样本数据值的分布情况quantile(φ, express)其中0 ≤ φ ≤ 1。

例如，当φ为0.5时，即表示找到当前样本数据中的中位数：
```
quantile(0.5, http_requests_total)
```


