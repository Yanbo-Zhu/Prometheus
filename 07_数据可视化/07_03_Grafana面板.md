

在第1章的“初始Prometheus”部分，我们已经带领读者大致了解了Grafana的基本使用方式。对于Grafana而言，Prometheus就是一个用于存储监控样本数据的数据源（Data Source）通过使用PromQL查询特定Prometheus实例中的数据并且在Panel中实现可视化。

接下来，我们将带领读者了解如何通过Panel创建精美的可视化图表。

# 1 添加新的Panel_设置数据源

Panel是Grafana中最基本的可视化单元。每一种类型的面板都提供了相应的查询编辑器(Query Editor)，让用户可以从不同的数据源（如Prometheus）中查询出相应的监控数据，并且以可视化的方式展现。

Grafana中所有的面板均以插件的形式进行使用，当前内置了5种类型的面板，分别是：Graph，Singlestat，Heatmap, Dashlist，Table以及Text。

其中像Graph这样的面板允许用户可视化任意多个监控指标以及多条时间序列。而Siglestat则必须要求查询结果为单个样本。Dashlist和Text相对比较特殊，它们与特定的数据源无关。

通过Grafana UI用户可以在一个Dashboard下添加Panel，点击Dashboard右上角的“Add Panel”按钮，如下所示，将会显示当前系统中所有可使用的Panel类型：

添加Panel
![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo1vpkWendubNJqvp%252Fgrafana_dashboard_add_panel.png%3Fgeneration%3D1540732503696105%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=c674fe93&sv=1)



选择想要创建的面板类型即可。这里以Graph面板为例，创建Panel之后，并切换到编辑模式，就可以进入Panel的配置页面。对于一个Panel而言，一般来说会包含2个主要的配置选项：General（通用设置）、Metrics（度量指标）。其余的配置则根据Panel类型的不同而不同。

在通用设置中，除了一些Panel的基本信息以外，最主要的能力就是定义动态Panel的能力，这部分内容会在本章的“模板化Dashboard”小结中详细介绍。

对于使用Prometheus作为数据源的用户，最主要的需要了解的就是Metrics设置的使用。

1 数据源
在Metric选项中可以定义了Grafana从哪些数据源中查询样本数据。
 **Data Source**中指定当前查询的数据源，Grafana会加载当前组织中添加的所有数据源。其中还会包含两个特殊的数据源：**Mixed**和**Grafana**。 Mixed用于需要从多个数据源中查询和渲染数据的场景，Grafana则用于需要查询Grafana自身状态时使用。

当选中数据源时，Panel会根据当前数据源类型加载不同的Query Editor界面。这里我们主要介绍Prometheus Query Editor，如下所示，当选中的数据源类型为Prometheus时，会显示如下界面：
![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo1vxA5nMrEawdH86%252Fgraph_prometheus_query_editor.png%3Fgeneration%3D1540732504063330%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=447fc563&sv=1)

2  PromQL 查询语句 
Grafana提供了对PromQL的完整支持，在Query Editor中，可以添加任意个Query，并且使用PromQL表达式从Prometheus中查询相应的样本数据。

```
avg (irate(node_cpu{mode!='idle'}[2m])) without (cpu)
```


3  **Legend format**
每个PromQL表达式都可能返回多条时间序列。**Legend format**用于控制如何格式化每条时间序列的图例信息。Grafana支持通过模板的方式，根据时间序列的标签动态生成图例名称，例如：使用表示使用当前时间序列中的instance标签的值作为图例名称：

```
{{instance}}-{{mode}}
```

当查询到的样本数据量非常大时可以导致Grafana渲染图标时出现一些性能问题，

4 Min Step 
通过**Min Step**可以控制Prometheus查询数据时的最小步长（Step），从而减少从Prometheus返回的数据量。


5 Resolution 
**Resolution**选项，则可以控制Grafana自身渲染的数据量。例如，如果**Resolution**的值为**1/10**，Grafana会将Prometeus返回的10个样本数据合并成一个点。因此**Resolution**越小可视化的精确性越高，反之，可视化的精度越低。


6 Format as 
**Format as**选项定义如何格式化Prometheus返回的样本数据。这里提供了3个选项：Table,Time Series和Heatmap，分别用于Tabel面板，Graph面板和Heatmap面板的数据可视化。


7 **Query Inspector**
除此以外，Query Editor还提供了调试相关的功能，点击**Query Inspector**可以展开相关的调试面板：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo1w0_yDF0Q8PvdWI%252Fgrafana_query_editor_inspector.png%3Fgeneration%3D1540732504205647%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=85822d57&sv=1)

在面板中，可以查看当前Prometheus返回的样本数据，用户也可以提供Mock数据渲染图像。




# 2 变化趋势：Graph面板


Graph面板是最常用的一种可视化面板，其通过折线图或者柱状图的形式显示监控样本随时间而变化的趋势。Graph面板天生适用于Prometheus中Gauge和Counter类型监控指标的监控数据可视化。例如，当需要查看主机CPU、内存使用率的随时间变化的情况时，可以使用Graph面板。同时，Graph还可以非常方便的支持多个数据之间的对比。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2-oeb4Ut2mdZ66Z%252Fgrafana_graph_panel.png%3Fgeneration%3D1540732501737332%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=34afd7d4&sv=1)

## 2.1 Graph面板与Prometheus

Graph面板通过折线图或者柱状图的形式，能够展示监控样本数据在一段时间内的变化趋势，因此其天生适合Prometheus中的Counter和Gauge类型的监控指标的可视化，对于Histogram类型的指标也可以支持，不过可视化效果不如Heatmap Panel来的直观。

## 2.2 使用Graph面板可视化Counter/Gauge

以主机为例，CPU使用率的变化趋势天然适用于使用Grapn面板来进行展示：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2-qKCnshDD1ktWb%252Fgrafana_graph_counter_demo_v2.png%3Fgeneration%3D1540732500633193%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=6cde658f&sv=1)

Prometheus Counter可视化


1 Metrics选项
在**Metrics选项**中，我们使用以下PromQL定义如何从Prometheus中读取数据：

```
1 - (avg(irate(node_cpu{mode='idle'}[5m])) without (cpu))
```

如下所示：
![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2-su92JHSfrluy-%252Fgrafana_graph_counter_demo_metrics.png%3Fgeneration%3D1540732501636165%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=2e2b7241&sv=1)



根据当前Prometheus的数据采集情况，该PromQL会返回多条时间序列（在示例中会返回3条）。Graph面板会从时间序列中获取样本数据，并绘制到图表中。 为了让折线图有更好的可读性，我们可以通过定义**Legend format**为`{{ instance }}`控制每条线的图例名称：

使用Legend format模板化图例
![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2-uMKm9w3Mv2yEE%252Fgrafana_graph_counter_demo_metrics_legend.png%3Fgeneration%3D1540732502836101%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=bd444010&sv=1)


---

2 **Axes选项**

由于当前使用的PromQL的数据范围为0~1表示CPU的使用率，为了能够更有效的表达出度量单位的概念，我们需要对Graph图表的坐标轴显示进行优化。如下所示，在中可以控制图标的X轴和Y轴相关的行为：
Axes选项
![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2-wzDDUki0a_CoF%252Fgrafana_graph_counter_demo_axes.png%3Fgeneration%3D1540732503295246%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=28a93c63&sv=1)


---

3 Legend选项

默认情况下，Y轴会直接显示当前样本的值，通过**Left Y**的**Unit**可以让Graph面板自动格式化样本值。当前表达式返回的当前主机CPU使用率的小数表示，因此，这里选择单位为**percent(0.0.-1.0)**。除了百分比以外，Graph面板支持如日期、货币、重量、面积等各种类型单位的自动换算，用户根据自己当前样本的值含义选择即可。

除了在Metrics设置图例的显示名称以外，在Graph面板的**Legend选项**可以进一步控制图例的显示方式，如下所示：
![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2-yBq8FxmrV2UzM%252Fgrafana_graph_counter_demo_legend.png%3Fgeneration%3D1540732502237889%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=60aab62a&sv=1)



**Options中**可以设置图例的显示方式以及展示位置，**Values**中可以设置是否显示当前时间序列的最小值，平均值等。 **Decimals**用于配置这些值显示时保留的小数位，如下所示：
Legend控制图例的显示示例
![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo20-X3uFBDuYUmAB%252Fgrafana_graph_counter_demo_legend_sample.png%3Fgeneration%3D1540732501603876%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=5887bd59&sv=1)


----

4 Display选项

除了以上设置以外，我们可能还需要对图表进行一些更高级的定制化，以便能够更直观的从可视化图表中获取信息。在Graph面板中**Display选项**可以帮助我们实现更多的可视化定制的能力，其中包含三个部分：Draw options、Series overrides和Thresholds。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo201TMHHxx8lHhVW%252Fgrafana_graph_counter_demo_display_draw.png%3Fgeneration%3D1540732502261859%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=4f1e289&sv=1)


1 **Draw Options**
**Draw Options**用于设置当前图标的展示形式、样式以及交互提示行为。其中，Draw Modes用于控制图形展示形式：Bar（柱状）、Lines（线条）、Points（点），用户可以根据自己的需求同时启用多种模式。Mode Options则设置各个展示模式下的相关样式。Hover tooltip用于控制当鼠标移动到图形时，显示提示框中的内容。

2 **Series overrides**
如果希望当前图表中的时间序列以不同的形式展示，则可以通过**Series overrides**控制，顾名思义，可以为指定的时间序列指定自定义的Draw Options配置，从而让其以不同的样式展示。例如：
Series overrides
![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo203au5c_bTpMycY%252Fgrafana_series_overrides.png%3Fgeneration%3D1540732501698840%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=19825f63&sv=1)


这里定义了一条自定义规则，其匹配图例名称满足**/localhost/**的时间序列，并定义其以点的形式显示在图表中，修改后的图标显示效果如下：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo205XgiYQUpbq4Zj%252Fgrafana_series_overrides_demo.png%3Fgeneration%3D1540732504829334%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=aefe93c1&sv=1)

Series overrides效果


3 **Thresholds**

Display选项中的最后一个是**Thresholds**，Threshold主要用于一些自定义一些样本的阈值，例如，定义一个Threshold规则，如果CPU超过50%的区域显示为warning状态，可以添加如下配置：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo207mjBBGvyKdbdI%252Fgrafana_thresholds_demo.png%3Fgeneration%3D1540732504578947%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=e0d7cb02&sv=1)

Threshold设置

Graph面板则会在图表中显示一条阈值，并且将所有高于该阈值的区域显示为warining状态，通过可视化的方式直观的在图表中显示一些可能出现异常的区域。

需要注意的是，如果用户为该图表自定义了Alert（告警）配置，Thresholds将会被警用，并且根据Alert中定义的Threshold在图形中显示阈值内容。关于Alert的使用会在后续部分，详细介绍。



## 2.3 使用Graph面板可视化Histogram

以Prometheus自身的监控指标prometheus_tsdb_compaction_duration为例，该监控指标记录了Prometheus进行数据压缩任务的运行耗时的分布统计情况。如下所示，是Prometheus返回的样本数据：

```
# HELP prometheus_tsdb_compaction_duration Duration of compaction runs.
# TYPE prometheus_tsdb_compaction_duration histogram
prometheus_tsdb_compaction_duration_bucket{le="1"} 2
prometheus_tsdb_compaction_duration_bucket{le="2"} 36
prometheus_tsdb_compaction_duration_bucket{le="4"} 36
prometheus_tsdb_compaction_duration_bucket{le="8"} 36
prometheus_tsdb_compaction_duration_bucket{le="16"} 36
prometheus_tsdb_compaction_duration_bucket{le="32"} 36
prometheus_tsdb_compaction_duration_bucket{le="64"} 36
prometheus_tsdb_compaction_duration_bucket{le="128"} 36
prometheus_tsdb_compaction_duration_bucket{le="256"} 36
prometheus_tsdb_compaction_duration_bucket{le="512"} 36
prometheus_tsdb_compaction_duration_bucket{le="+Inf"} 36
prometheus_tsdb_compaction_duration_sum 51.31017077500001
prometheus_tsdb_compaction_duration_count 36
```

在第2章的“Metric类型”小节中，我们已经介绍过Histogram的指标，Histogram用于统计样本数据的分布情况，其中标签le定义了分布桶Bucket的边界，如上所示，表示当前Prometheus共进行了36次数据压缩，总耗时为51.31017077500001ms。其中任务耗时在0~1ms区间内的为2次、在0~2ms区间范围内为36次，以此类推。

如下所示，如果需要在Graph中显示Histogram类型的监控指标，需要在Query Editor中定义查询结果的**Format as**为Heatmap。通过该设置Grafana会自动计算Histogram中的Bucket边界范围以及该范围内的值：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo209nnpwYQ-QjYiy%252Fgrafana_bucket_setting.png%3Fgeneration%3D1540732503297771%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=241dae7c&sv=1)

Metrics设置

Graph面板重新计算了Bucket边界，如下所示，在0~1ms范围内的任务次数为2，在1~2ms范围内的运行任务次数为34。通过图形的面积，可以反映出各个Bucket下的大致数据分布情况：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo20BtHQRPXmKRg0B%252Fgrafana_bucket_demo.png%3Fgeneration%3D1540732505570853%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=96c790e5&sv=1)


不过通过Graph面板展示Histogram也并不太直观，其并不能直接反映出Bucket的大小以及分布情况，因此在Grafana V5版本以后更推荐使用Heatmap面板的方式展示Histogram样本数据。关于Heatmap面板的使用将会在接下来的部分介绍。


# 3 分布统计_Heatmap面板


Heatmap是是Grafana v4.3版本以后新添加的可视化面板，通过热图可以直观的查看样本的分布情况。在Grafana v5.1版本中Heatmap完善了对Prometheus的支持。这部分，将介绍如何使用Heatmap Panel实现对Prometheus监控指标的可视化。

## 3.1 使用Heatmap可视化Histogram样本分布情况

在上一小节中，我们尝试了使用Graph面板来可视化Histogram类型的监控指标prometheus_tsdb_compaction_duration_bucket。虽然能展示各个Bucket区间内的样本分布，但是无论是以线图还是柱状图的形式展示，都不够直观。对于Histogram类型的监控指标来说，更好的选择是采用Heatmap Panel，如下所示，Heatmap Panel可以自动对Histogram类型的监控指标分布情况进行计划，获取到每个区间范围内的样本个数，并且以颜色的深浅来表示当前区间内样本个数的大小。而图形的高度，则反映出当前时间点，样本分布的离散程度。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2E3RI496WdVE5mb%252Fgrafana_heatmap_sample.png%3Fgeneration%3D1540732505610362%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=39818cca&sv=1)


在Grafana中使用Heatmap Panel也非常简单，在Dashboard页面右上角菜单中点击“add panel”按钮，并选择Heatmap Panel即可。

如下所示，Heapmap Panel的编辑页面中，主要包含5类配置选项，分别是：General、Metrics、Axes、Display、Time range。

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2E60MZ8UWeUSSWG%252Fgrafana_heatmap_editor.png%3Fgeneration%3D1540732505063564%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=c8ddf31d&sv=1)

Heapmap Panel编辑页面

其中大部分的配置选项与Graph面板基本保持一致，这里就不重复介绍了。


1 Mteircs设置

当使用Heatmap可视化Histogram类型的监控指标时，需要设置**Format as**选项为**Heatmap**。当使用Heatmap格式化数据后，Grafana会自动根据样本的中的le标签，计算各个Bucket桶内的分布，并且按照Bucket对数据进行重新排序。**Legend format**模板则将会控制Y轴中的显示内容。如下所示：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2E8_jySm_j31Eop%252Fgrafana_heatmap_metrics_setting.png%3Fgeneration%3D1540732507437004%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=fd6c754&sv=1)


---

 2 **Axes选项**
 
默认情况下，Heatmap Panel会自行对PromQL查询出的数据进行分布情况统计，而在Prometheus中Histogram类型的监控指标其实是已经自带了分布的Bucket信息的，因此为了直接使用这些Bucket信息，我们需要在**Axes选项**中定义数据的Date format需要定义为**Time series buckets**。该选项表示Heatmap Panel不需要自身对数据的分布情况进行计算，直接使用时间序列中返回的Bucket即可。如下所示：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2EA71TbrRqowNGM%252Fgrafana_heatmap_axes_setting.png%3Fgeneration%3D1540732505123456%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=e1bfea00&sv=1)



通过以上设置，即可实现对Histogram类型监控指标的可视化。

 

## 3.2 使用Heatmap可视化其它类型样本分布情况

对于非Histogram类型，由于其监控样本中并不包含Bucket相关信息，因此在**Metrics选项中**需要定义**Format as**为**Time series**，如下所示：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2EC_0SU4EBOdC89%252Fgrafana_heatmap_normal_metrics.png%3Fgeneration%3D1540732504572275%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=448d2aa9&sv=1)


并且通过**Axes选项**中选择**Data format**方式为**Time series**。设置该选项后Heatmap Panel会要求用户提供Bucket分布范围的设置，如下所示：

Axes设置
![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2EH0tFvSzG1aQOl%252Fgrafana_heatmap_normal_axes.png%3Fgeneration%3D1540732504568893%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=dda4b60&sv=1)


在Y轴（Y Axis）中需要通过Scale定义Bucket桶的分布范围，默认的Bucket范围支持包括：liner（线性分布）、log(base 10)（10的对数）、log(base 32)（32的对数）、log(base 1024)（1024的对数）等。

例如，上图中设置的Scale为log(base 2)，那么在Bucket范围将2的对数的形式进行分布，即[1,2,4,8,....]，如下所示：

Bucket分布情况
![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2EJdL2_tHYhaiHj%252Fgrafana_heatmap_normal_sample.png%3Fgeneration%3D1540732500006872%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=ec5939&sv=1)


通过以上设置，Heatmap会自动根据用户定义的Bucket范围对Prometheus中查询到的样本数据进行分布统计。



# 4 当前状态_SingleStat面板


Singlem Panel侧重于展示系统的当前状态而非变化趋势。如下所示，在以下场景中特别适用于使用SingleStat：
- 当前系统中所有服务的运行状态；
- 当前基础设施资源的使用量；
- 当前系统中某些事件发生的次数或者资源数量等。

如下所示，是使用SingleStat进行数据可视化的显示效果：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2E-efMb6CoyKJwN%252Fgrafana_singlestat_sample.png%3Fgeneration%3D1540732506685133%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=1e0297d0&sv=1)

SingleStat Panel示例



## 4.1 使用SingleStat Panel



1 metrics
从Dashboardc创建Singlestat Panel，并进入编辑页面， 如下所示：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2E2MBdAqkYchNKf%252Fgrafana_single_stat_sample.png%3Fgeneration%3D1540732505629848%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=fa562f0&sv=1)


对于SingleStat Panel而言，其只能处理一条时间序列，否则页面中会提示“Multiple Series Error”错误信息。这里使用如下PromQL查询当前主机负载：

```
node_load1{instance="localhost:9100"}
```

默认情况下，当前面板中会显示当前时间序列中所有样本的平均值，而实际情况下，我们需要显示的是当前主机当前的负载情况，因此需要通过SingleStat Panel的**Options**选项控制当前面板的显示模式：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2E5yCe5QnI3l6UA%252Fgrafana_single_stat_edit_options.png%3Fgeneration%3D1540732506407638%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=7e5b4980&sv=1)


---

2  SingleStat Option选项

如上所示，通过Value配置项组可以控制当前面板中显示的值，以及字体大小等。对于主机负载而言，我们希望能够显示当前的最新值，因此修改Stat为**Current**即可。

如果希望面板能够根据不同的值显示不同的颜色的话，则可以定义**Thresholds**与**Colors**的映射关系，例如，定义Thresholds的分割区间值为“0,1”，则当Value的值落到不同的范围内时，将显示不同的颜色。

如果希望能够显示当前时间序列的样本值变化情况，则可以启用Spark lines配置。启用之后，Singlestat面板中除了会显示当前的最新样本值以外，也会同时将时间序列中的数据已趋势图的形式进行展示。

除了通过数字大小反应当前状态以外，在某些场景下我们可能更关心的是这些数字表示的意义。例如，在Promthues监控服务的健康状态时，在样本数据中会通过0表示不健康，1表示健康。 但是如果直接将0或1显示在面板中，那么可视化效果将缺乏一定的可读性。

---

3 **Value Mappings**

为了提升数字的可读性，可以在Singlestat Panel中可以通过**Value Mappings**定义值的映射关系。Siglesta支持值映射（value to text）和区间映射（range to text）两种方式。 如下所示：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2E7h0szdERrvexJ%252Fgrafana_single_stat_edit_value_mapping.png%3Fgeneration%3D1540732506205193%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=29e653ee&sv=1)

Singlestat value mappings配置

当面板中Value的值在0~0.99范围内则显示为Health，否则显示为Unhealth。这种模式特别适合于展示服务的健康状态。 当然你也可以将Value映射为任意的字符，甚至是直接使用Emoji([http://www.iemoji.com/](http://www.iemoji.com/))表情：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPuo-z7M1wVtmm-6lNl%252F-LPuo2E9YWsQoux858y4%252Fgrafana_single_stat_edit_value_mapping_emoji.png%3Fgeneration%3D1540732500212409%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=b9bdd38c&sv=1)


