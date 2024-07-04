

使用PromQL除了能够方便的按照查询和过滤时间序列以外，PromQL还支持丰富的操作符，用户可以使用这些操作符对进一步的对事件序列进行二次加工。这些操作符包括：数学运算符，逻辑运算符，布尔运算符等等。

```
Prometheus 查询语句中，支持常见的各种表达式操作符，例如

算术运算符：支持的算术运算符有 +，-，*，/，%，^, 例如 http_requests_total * 2 表示将 http_requests_total 所有数据 double 一倍。

比较运算符：支持的比较运算符有 ==，!=，>，<，>=，<=, 例如 http_requests_total > 100 表示 http_requests_total 结果中大于 100 的数据。

逻辑运算符：支持的逻辑运算符有 and，or，unless(排除), 例如 http_requests_total == 5 or http_requests_total == 2 表示 http_requests_total 结果中等于 5 或者 2 的数据。

聚合运算符：支持的聚合运算符有 sum，min，max，avg，stddev，stdvar，count，count_values，bottomk，topk，quantile，, 例如 max(http_requests_total) 表示 http_requests_total 结果中最大的数据。
```

# 1 总览

操作符Operators，是针对Instant vector和Scalar之间的运算，不能用于Range vector：
```
node_load1 * 100
{instance="host.docker.internal:9100", job="node0"} 221.09375
{instance="host.docker.internal:9102", job="node2"} 28.000
```



对两个Instant vector相除时，他们的label要相同，比如计算网络包所占的字节数：
```
node_network_receive_bytes_total{device="en0"}/node_network_receive_packets_total
{device="en0", instance=":9100", job="node0"} 17.207300663986636
```

> Note：如果label不同就没有结果。计算结果会丢弃metric的名称，只保留匹配的label。




也支持集合的操作，比如or将两个vector合并了，比如把load和cpu展示在一个图：
```
node_load1 or node_cpu_seconds_total{cpu="0",mode="user"}
node_load1{job="node0"} 3.33447265625
node_load1{job="node2"} 0.66
node_cpu_seconds_total{cpu="0", job="node0", mode="user"} 48824.2
node_cpu_seconds_total{cpu="0", job="node2", mode="user"} 1115.41
```

> Note: 这样可以把两个指标展示在一个图中了，同时对不同的指标进行处理。是完全按label匹配，而不是按值匹配，因为两个指标的值几乎是不会相等。


# 2 数学运算

例如，我们可以通过指标node_memory_free_bytes_total获取当前主机可用的内存空间大小，其样本单位为Bytes。这是如果客户端要求使用MB作为单位响应数据，那只需要将查询到的时间序列的样本值进行单位换算即可：

```
node_memory_free_bytes_total / (1024 * 1024)
```

node_memory_free_bytes_total表达式会查询出所有满足表达式条件的时间序列，在上一小节中我们称该表达式为瞬时向量表达式，而返回的结果成为瞬时向量。

当瞬时向量与标量之间进行数学运算时，数学运算符会依次作用域瞬时向量中的每一个样本值，从而得到一组新的时间序列。

而如果是瞬时向量与瞬时向量之间进行数学运算时，过程会相对复杂一点。 例如，如果我们想根据node_disk_bytes_written和node_disk_bytes_read获取主机磁盘IO的总量，可以使用如下表达式：

```
node_disk_bytes_written + node_disk_bytes_read
```

那这个表达式是如何工作的呢？==依次找到与左边向量元素匹配（标签完全一致）的右边向量元素进行运算，如果没找到匹配元素，则直接丢弃。同时新的时间序列将不会包含指标名称。==

该表达式返回结果的示例如下所示：
```
{device="sda",instance="localhost:9100",job="node_exporter"}=>1634967552@1518146427.807 + 864551424@1518146427.807
{device="sdb",instance="localhost:9100",job="node_exporter"}=>0@1518146427.807 + 1744384@1518146427.807
```

PromQL支持的所有数学运算符如下所示：
- `+` (加法)
- `-` (减法)
- `*` (乘法)
- `/` (除法)
- `%` (求余)
- `^` (幂运算)




# 3 使用布尔运算过滤时间序列_比较运算符

在PromQL通过标签匹配模式，用户可以根据时间序列的特征维度对其进行查询。
而布尔运算则支持用户根据时间序列中样本的值，对时间序列进行过滤。

例如，通过数学运算符我们可以很方便的计算出，当前所有主机节点的内存使用率：
```
(node_memory_bytes_total - node_memory_free_bytes_total) / node_memory_bytes_total
```


而系统管理员在排查问题的时候可能只想知道当前内存使用率超过95%的主机呢？通过使用布尔运算符可以方便的获取到该结果：
```
(node_memory_bytes_total - node_memory_free_bytes_total) / node_memory_bytes_total > 0.95
```



瞬时向量与标量进行布尔运算时，PromQL依次比较向量中的所有时间序列样本的值，如果比较结果为true则保留，反之丢弃。


瞬时向量与瞬时向量直接进行布尔运算时，同样遵循默认的匹配模式：依次找到与左边向量元素匹配（标签完全一致）的右边向量元素进行相应的操作，如果没找到匹配元素，则直接丢弃。

目前，Prometheus支持以下布尔运算符如下：

- `==` (相等)
- `!=` (不相等)
- `>` (大于)
- `<` (小于)
- `>=` (大于等于)
- `<=` (小于等于)



# 4 使用bool修饰符改变布尔运算符的行为


布尔运算符的默认行为是对时序数据进行过滤。而在其它的情况下我们可能需要的是真正的布尔结果。例如，只需要知道当前模块的HTTP请求量是否>=1000，如果大于等于1000则返回1（true）否则返回0（false）。这时可以使用bool修饰符改变布尔运算的默认行为。 例如：

```
http_requests_total > bool 1000
```

使用bool修改符后，布尔运算不会对时间序列进行过滤，而是直接依次瞬时向量中的各个样本数据与标量的比较结果0或者1。从而形成一条新的时间序列。

```
http_requests_total{code="200",handler="query",instance="localhost:9090",job="prometheus",method="get"}  1
http_requests_total{code="200",handler="query_range",instance="localhost:9090",job="prometheus",method="get"}  0
```

同时需要注意的是，如果是在两个标量之间使用布尔运算，则必须使用bool修饰符

```
2 == bool 2 # 结果为1
```


# 5 使用集合运算符

使用瞬时向量表达式能够获取到一个包含多个时间序列的集合，我们称为瞬时向量。 通过集合运算，可以在两个瞬时向量与瞬时向量之间进行相应的集合操作。目前，Prometheus支持以下集合运算符：
- `and` (并且)
- `or` (或者)
- `unless` (排除)

_**vector1 and vector2**_ 会产生一个由vector1的元素组成的新的向量。该向量包含vector1中完全匹配vector2中的元素组成。

_**vector1 or vector2**_ 会产生一个新的向量，该向量包含vector1中所有的样本数据，以及vector2中没有与vector1匹配到的样本数据。

_**vector1 unless vector2**_ 会产生一个新的向量，新向量中的元素由vector1中没有与vector2匹配的元素组成。



# 6 操作符优先级

对于复杂类型的表达式，需要了解运算操作的运行优先级

例如，查询主机的CPU使用率，可以使用表达式：

```
100 * (1 - avg (irate(node_cpu{mode='idle'}[5m])) by(job) )
```

其中irate是PromQL中的内置函数，用于计算区间向量中时间序列每秒的即时增长率。关于内置函数的部分，会在下一节详细介绍。

在PromQL操作符中优先级由高到低依次为：
1. `^`
2. `*, /, %`
3. `+, -`
4. `==, !=, <=, <, >=, >`
5. `and, unless`
6. `or`


