

使用PromQL除了能够方便的按照查询和过滤时间序列以外，PromQL还支持丰富的操作符，用户可以使用这些操作符对进一步的对事件序列进行二次加工。这些操作符包括：数学运算符，逻辑运算符，布尔运算符等等。

```
Prometheus 查询语句中，支持常见的各种表达式操作符，例如

算术运算符：支持的算术运算符有 +，-，*，/，%，^, 例如 http_requests_total * 2 表示将 http_requests_total 所有数据 double 一倍。

比较运算符：支持的比较运算符有 ==，!=，>，<，>=，<=, 例如 http_requests_total > 100 表示 http_requests_total 结果中大于 100 的数据。

逻辑运算符：支持的逻辑运算符有 and，or，unless(排除), 例如 http_requests_total == 5 or http_requests_total == 2 表示 http_requests_total 结果中等于 5 或者 2 的数据。

聚合运算符：支持的聚合运算符有 sum，min，max，avg，stddev，stdvar，count，count_values，bottomk，topk，quantile，, 例如 max(http_requests_total) 表示 http_requests_total 结果中最大的数据。
```

# 1 操作符优先级

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



# 2 总览

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


# 3 数学运算

PromQL支持的所有数学运算符如下所示：
- `+` (加法)
- `-` (减法)
- `*` (乘法)
- `/` (除法)
- `%` (求余)
- `^` (幂运算)


二元运算操作符支持 scalar/scalar(标量/标量)、vector/scalar(向量/标量)、和 vector/vector(向量/向量) 之间的操作。


例如，我们可以通过指标node_memory_free_bytes_total获取当前主机可用的内存空间大小，其样本单位为Bytes。这是如果客户端要求使用MB作为单位响应数据，那只需要将查询到的时间序列的样本值进行单位换算即可：

```
node_memory_free_bytes_total / (1024 * 1024)
```

node_memory_free_bytes_total表达式会查询出所有满足表达式条件的时间序列，在上一小节中我们称该表达式为瞬时向量表达式，而返回的结果成为瞬时向量。




1 
在两个标量之间进行数学运算，得到的结果也是标量。


---


2 
在向量和标量之间，这个运算符会作用于这个向量的每个样本值上。例如：如果一个时间序列瞬时向量除以 2，操作结果也是一个新的瞬时向量，且度量指标名称不变, 它是原度量指标瞬时向量的每个样本值除以 2。

当瞬时向量与标量之间进行数学运算时，数学运算符会依次作用域瞬时向量中的每一个样本值，从而得到一组新的时间序列。

---


3 
如果是瞬时向量与瞬时向量之间进行数学运算时，过程会相对复杂一点，运算符会依次找到与左边向量元素匹配（标签完全一致）的右边向量元素进行运算，如果没找到匹配元素，则直接丢弃。同时新的时间序列将不会包含指标名称。



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





# 4 使用布尔运算过滤时间序列_比较运算符


- `==` (相等)
- `!=` (不相等)
- `>` (大于)
- `<` (小于)
- `>=` (大于等于)
- `<=` (小于等于)

布尔运算符被应用于 `scalar/scalar（标量/标量）`、`vector/scalar（向量/标量）`，和`vector/vector（向量/向量）`。默认情况下布尔运算符只会根据时间序列中样本的值，对时间序列进行过滤。我们可以通过在运算符后面使用 `bool` 修饰符来改变布尔运算的默认行为。使用 bool 修改符后，布尔运算不会对时间序列进行过滤，而是直接依次瞬时向量中的各个样本数据与标量的比较结果 `0` 或者 `1`。

----

在两个标量之间进行布尔运算，必须提供 bool 修饰符，得到的结果也是标量，即 `0`（`false`）或 `1`（`true`）。例如：
2 > bool 1 # 结果为 1

---

瞬时向量和标量之间的布尔运算，这个运算符会应用到某个当前时刻的每个时序数据上，如果一个时序数据的样本值与这个标量比较的结果是 `false`，则这个时序数据被丢弃掉，如果是 `true`, 则这个时序数据被保留在结果中。如果提供了 bool 修饰符，那么比较结果是 `0` 的时序数据被丢弃掉，而比较结果是 `1` 的时序数据被保留。


瞬时向量与标量进行布尔运算时，PromQL依次比较向量中的所有时间序列样本的值，如果比较结果为true则保留，反之丢弃。

----

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

----

时向量与瞬时向量直接进行布尔运算时，同样遵循默认的匹配模式：依次找到与左边向量元素匹配（标签完全一致）的右边向量元素进行相应的操作，如果没找到匹配元素，或者计算结果为 false，则直接丢弃。如果匹配上了，则将左边向量的度量指标和标签的样本数据写入瞬时向量。如果提供了 bool 修饰符，那么比较结果是 0 的时序数据被丢弃掉，而比较结果是 1 的时序数据（只保留左边向量）被保留。


瞬时向量与瞬时向量直接进行布尔运算时，同样遵循默认的匹配模式：依次找到与左边向量元素匹配（标签完全一致）的右边向量元素进行相应的操作，如果没找到匹配元素，则直接丢弃。



# 5 使用bool修饰符改变布尔运算符的行为


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


# 6 使用集合运算符

使用瞬时向量表达式能够获取到一个包含多个时间序列的集合，我们称为瞬时向量。 通过集合运算，可以在两个瞬时向量与瞬时向量之间进行相应的集合操作。目前，Prometheus支持以下集合运算符：
- `and` (并且)
- `or` (或者)
- `unless` (排除)

_**vector1 and vector2**_ 会产生一个由vector1的元素组成的新的向量。该向量包含vector1中完全匹配vector2中的元素组成。

_**vector1 or vector2**_ 会产生一个新的向量，该向量包含vector1中所有的样本数据，以及vector2中没有与vector1匹配到的样本数据。

_**vector1 unless vector2**_ 会产生一个新的向量，新向量中的元素由vector1中没有与vector2匹配的元素组成。




