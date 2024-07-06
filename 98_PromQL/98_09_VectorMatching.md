
https://www.cnblogs.com/JetpropelledSnake/p/16299415.html

向量与向量之间进行运算操作时会基于默认的匹配规则：依次找到与左边向量元素匹配（标签完全一致）的右边向量元素进行运算，如果没找到匹配元素，则直接丢弃。

接下来将介绍在PromQL中有两种典型的匹配模式：一对一（one-to-one）,多对一（many-to-one）或一对多（one-to-many）。

向量与向量之间进行运算操作时会基于默认的匹配规则：依次找到与左边向量元素匹配（标签完全一致）的右边向量元素进行运算，如果没找到匹配元素，则直接丢弃。

接下来将介绍在 PromQL 中有两种典型的匹配模式：一对一（one-to-one）,多对一（many-to-one）或一对多（one-to-many）。



# 1 一对一匹配

一对一 匹配模式会从操作符两边表达式获取的瞬时向量依次比较并找到唯一匹配(标签完全一致)的样本值。默认情况下，使用表达式：
```
vector1 <operator> vector2
```

在操作符两边表达式标签不一致的情况下，可以使用on(label list)或者ignoring(label list）来修改便签的匹配行为。
==使用ignoreing可以在匹配时忽略某些便签，就是就当这个标签不存在了， 这个标签的值也不考虑了。==
==而on则用于将匹配行为限定在某些便签之内==
```
<vector expr> <bin-op> ignoring(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) <vector expr>
```

例如当存在样本：
```
method_code:http_errors:rate5m{method="get", code="500"}  24
method_code:http_errors:rate5m{method="get", code="404"}  30
method_code:http_errors:rate5m{method="put", code="501"}  3
method_code:http_errors:rate5m{method="post", code="500"} 6
method_code:http_errors:rate5m{method="post", code="404"} 21

method:http_requests:rate5m{method="get"}  600
method:http_requests:rate5m{method="del"}  34
method:http_requests:rate5m{method="post"} 120
```

使用PromQL表达式：
```
method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
```

该表达式会返回在过去5分钟内，HTTP请求状态码为500的在所有请求中的比例。如果没有使用ignoring(code)，操作符两边表达式返回的瞬时向量中将找不到任何一个标签完全相同的匹配项。


因此结果如下：
```
{method="get"}  0.04            //  24 / 600
{method="post"} 0.05            //   6 / 120
```

同时由于method为put和del的样本找不到匹配项，因此不会出现在结果当中。


![](https://img2022.cnblogs.com/blog/1354564/202205/1354564-20220522224145816-1961346373.png)

# 2 多对一和一对多

多对一和一对多两种匹配模式指的是“一”侧的每一个向量元素可以与"多"侧的多个元素匹配的情况。在这种情况下，必须使用group修饰符：group_left或者group_right来确定哪一个向量具有更高的基数（充当“多”的角色）。

```
<vector expr> <bin-op> ignoring(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> ignoring(<label list>) group_right(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_left(<label list>) <vector expr>
<vector expr> <bin-op> on(<label list>) group_right(<label list>) <vector expr>
```

多对一和一对多两种模式一定是出现在操作符两侧表达式返回的向量标签不一致的情况。因此需要使用ignoring和on修饰符来排除或者限定匹配的标签列表。

例如,使用表达式：

```
method_code:http_errors:rate5m / ignoring(code) group_left method:http_requests:rate5m
```

该表达式中，左向量`method_code:http_errors:rate5m`包含两个标签method和code。而右向量`method:http_requests:rate5m`中只包含一个标签method，因此匹配时需要使用ignoring限定匹配的标签为code。 
在限定匹配标签后，右向量中的元素 (method:http_requests:rate5m) 可能匹配到多个左向量中的元素 因此该表达式的匹配模式为多对一，==所以多的那一侧 需要被 group==,  需要使用group修饰符group_left指定左向量具有更好的基数。



最终的运算结果如下：

```
{method="get", code="500"}  0.04            //  24 / 600
{method="get", code="404"}  0.05            //  30 / 600
{method="post", code="500"} 0.05            //   6 / 120
{method="post", code="404"} 0.175           //  21 / 120
```

> 提醒：group修饰符只能在比较和数学运算符中使用。在逻辑运算and,unless和or才注意操作中默认与右向量中的所有元素进行匹配。


![](https://img2022.cnblogs.com/blog/1354564/202205/1354564-20220522224304185-545664108.png)



# 3 多对多


![](https://img2022.cnblogs.com/blog/1354564/202205/1354564-20220522224324815-1108801453.png)

分类: [Prometheus监控学习笔记](https://www.cnblogs.com/JetpropelledSnake/category/1359803.html)


# 4 #


Vector matching向量匹配，针对两个可能不完全相等label的向量，比如node比cpu_seconds就少了cpu和mode标签：
```
node_load1{job="node0"} 3.33447265625
node_load1{job="node2"} 0.66
node_cpu_seconds_total{cpu="0", job="node0", mode="user"} 48824.2
node_cpu_seconds_total{cpu="0", job="node2", mode="user"} 1115.41
```



如果直接把两个相除，则是空，因为标签并不匹配，无法相除，这时候可以忽略掉cpu和mode，这就是一对一的匹配：
```
node_cpu_seconds_total{cpu="0",mode="user"} / ignoring(cpu,mode) node_load1
{instance="host.docker.internal:9100", job="node0"} 17082.595555555556
{instance="host.docker.internal:9102", job="node2"} 19009.5
```



如果我们选择两个cpu，即cpu0和1，这时候就是多对一的关系：
```
node_load1 or node_cpu_seconds_total{cpu=~"0|1",mode="user"}
node_load1{instance="host.docker.internal:9100", job="node0"} 2.92138671875
node_load1{instance="host.docker.internal:9102", job="node2"} 0.13
node_cpu_seconds_total{cpu="0", instance="host.docker.internal:9100", job="node0", mode="user"} 48954.11
node_cpu_seconds_total{cpu="0", instance="host.docker.internal:9102", job="node2", mode="user"} 1141.35
node_cpu_seconds_total{cpu="1", instance="host.docker.internal:9100", job="node0", mode="user"} 8483.03
node_cpu_seconds_total{cpu="1", instance="host.docker.internal:9102", job="node2", mode="user"} 1148.34
```




针对多对一的匹配运算，就不能直接相除了，而是需要加上group_left，即以左边的为基数：

node_cpu_seconds_total{cpu=~"0|1",mode="user"} / ignoring(cpu,mode) group_left node_load1
{cpu="0", instance=":9100", job="node0", mode="user"} 19395.831582205028
{cpu="0", instance=":9102", job="node2", mode="user"} 12694.333333333334
{cpu="1", instance=":9100", job="node0", mode="user"} 3360.5223984526115
{cpu="1", instance=":9102", job="node2", mode="user"} 12772.555555555557



使用的是ignore某些标签让剩下的标签相等，也可以on直接指定匹配的标签，结果也是一样的：

node_cpu_seconds_total{cpu=~"0|1",mode="user"} / on(instance) group_left node_load1

