
Console Teamplet虽然能满足一定的可视化需求，但是也仅仅是对Prometheus的基本能力的补充。同时使用也会有许多问题，首先用户需要学习和了解Go Template模板语言，其它其支持的可视化图表类型也非常有限，最后其管理也有一定的成本。在第1章的“初识Prometheus”中我们已经尝试通过Grafana快速搭建过一个主机监控的Dashboard，在本章中将会带来读者学习如何使用Grafana创建更加精美的可视化报表。


# 1 概况
## 1.1 1.使用：

- 配置方便：支持Dashboard、Panel、Row等组合，且支持折线图、柱状图等多种图例
- 图表漂亮：可以选择暗黑系或纯白系，你也可以自己定义颜色
- 模板很多：grafana模板很活跃，有很多[用户贡献的面板](https://grafana.com/dashboards)，直接导入就能用
- 支持多种数据源：grafana作为展示面板，支持很多数据源，如Graphite、ES、Prometheus等
- 权限管理简单：有admin、viewer等多种角色管控

## 1.2 2.二次开发：

如果默认的grafana不能满足你的需求，要二次开发，官方也提供了很多支持：

- 协议为Apache License 2.0：商业友好，随便改吧，改完拿去卖也行。
- 完善的API调用：权限、面板、用户、报警都支持api调用。
- 多种鉴权方式：OAuth、LADP、Proxy多种方式，你可以接入自己公司的鉴权系统
- 插件开发：如果你不想直接改代码，可以做自己的插件
- go+Angular+react：常用的技术栈，方便二次开发

prometheus + grafana 做为监控组合很方便，很强大，改造了鉴权之后更加香。一开始我们还尝试使用grafana自带的报警功能，可惜比较鸡肋，无法用于生产，报警的issue一大堆官方也不想修改，作罢

# 2 Grafana基本概念

首先Grafana是一个通用的可视化工具。‘通用’意味着Grafana不仅仅适用于展示Prometheus下的监控数据，也同样适用于一些其他的数据可视化需求。在开始使用Grafana之前，我们首先需要明确一些Grafana下的基本概念，以帮助用户能够快速理解Grafana。


## 2.1 数据源（Data Source）

对于Grafana而言，Prometheus这类为其提供数据的对象均称为数据源（Data Source）。目前，Grafana官方提供了对：Graphite, InfluxDB, OpenTSDB, Prometheus, Elasticsearch, CloudWatch的支持。对于Grafana管理员而言，只需要将这些对象以数据源的形式添加到Grafana中，Grafana便可以轻松的实现对这些数据的可视化工作。


# 3 仪表盘（Dashboard）

通过数据源定义好可视化的数据来源之后，对于用户而言最重要的事情就是实现数据的可视化。在Grafana中，我们通过Dashboard来组织和管理我们的数据可视化图表：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LRX7f16nDH_eSWU4Szc%252F-LRX7gzvWm8jDHg4S9fc%252Fdashboard-components.png%3Fgeneration%3D1542465969279644%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=68f478d7&sv=1)



如上所示，在一个Dashboard中一个最基本的可视化单元为一个**Panel（面板）**，Panel通过如趋势图，热力图的形式展示可视化数据。 并且在Dashboard中每一个Panel是一个完全独立的部分，通过Panel的**Query Editor（查询编辑器）**我们可以为每一个Panel自己查询的数据源以及数据查询方式，
例如，如果以Prometheus作为数据源，那在Query Editor中，我们实际上使用的是PromQL，而Panel则会负责从特定的Prometheus中查询出相应的数据，并且将其可视化。由于每个Panel是完全独立的，因此在一个Dashboard中，往往可能会包含来自多个Data Source的数据。

Grafana通过插件的形式提供了多种Panel的实现，常用的如：Graph Panel，Heatmap Panel，SingleStat Panel以及Table Panel等。用户还可通过插件安装更多类型的Panel面板。

除了Panel以外，在Dashboard页面中，我们还可以定义一个**Row（行）**，来组织和管理一组相关的Panel。

除了Panel, Row这些对象以外，Grafana还允许用户为Dashboard定义**Templating variables（模板参数）**，从而实现可以与用户动态交互的Dashboard页面。同时Grafana通过JSON数据结构管理了整个Dasboard的定义，因此这些Dashboard也是非常方便进行共享的。Grafana还专门为Dashboard提供了一个共享服务：[https://grafana.com/dashboards](https://grafana.com/dashboards)，通过该服务用户可以轻松实现Dashboard的共享，同时我们也能快速的从中找到我们希望的Dashboard实现，并导入到自己的Grafana中。



# 4 组织和用户

作为一个通用可视化工具，Grafana除了提供灵活的可视化定制能力以外，还提供了面向企业的组织级管理能力。在Grafana中Dashboard是属于一个**Organization（组织）**，通过Organization，可以在更大规模上使用Grafana，例如对于一个企业而言，我们可以创建多个Organization，其中**User（用户）**可以属于一个或多个不同的Organization。 并且在不同的Organization下，可以为User赋予不同的权限。 从而可以有效的根据企业的组织架构定义整个管理模型。



# 5 例子




Prometheus UI提供了快速验证PromQL以及临时可视化支持的能力，而在大多数场景下引入监控系统通常还需要构建可以长期使用的监控数据可视化面板（Dashboard）。这时用户可以考虑使用第三方的可视化工具如Grafana，


Grafana是一个开源的可视化平台，并且提供了对Prometheus的完整支持。

```
docker run -d -p 3000:3000 grafana/grafana
```

访问[http://localhost:3000](http://localhost:3000)就可以进入到Grafana的界面中，默认情况下使用账户admin/admin进行登录。在Grafana首页中显示默认的使用向导，包括：安装、添加数据源、创建Dashboard、邀请成员、以及安装应用和插件等主要流程:

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPMFlGDFIX7wuLhSHx9%252F-LPMFp733ZL0hekE5YPo%252Fget_start_with_grafana2.png%3Fgeneration%3D1540136069592465%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=68ebe28b&sv=1)




-----

这里将添加Prometheus作为默认的数据源，如下图所示，指定数据源类型为Prometheus并且设置Prometheus的访问地址即可，在配置正确的情况下点击“Add”按钮，会提示连接成功的信息：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPMFlGDFIX7wuLhSHx9%252F-LPMFp75TqWkguvVkIKH%252Fadd_default_prometheus_datasource.png%3Fgeneration%3D1540136069024088%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=21d96e56&sv=1)

添加Prometheus作为数据源


-----


在完成数据源的添加之后就可以在Grafana中创建我们可视化Dashboard了。Grafana提供了对PromQL的完整支持，如下所示，通过Grafana添加Dashboard并且为该Dashboard添加一个类型为“Graph”的面板。 并在该面板的“Metrics”选项下通过PromQL查询需要可视化的数据：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPMFlGDFIX7wuLhSHx9%252F-LPMFp770M-VxTvT4XU4%252Ffirst_grafana_dashboard.png%3Fgeneration%3D1540136067470329%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=c31eb484&sv=1)

第一个可视化面板


---

点击界面中的保存选项，就创建了我们的第一个可视化Dashboard了。 当然作为开源软件，Grafana社区鼓励用户分享Dashboard通过[https://grafana.com/dashboards](https://grafana.com/dashboards)网站，可以找到大量可直接使用的Dashboard：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPMFlGDFIX7wuLhSHx9%252F-LPMFp79QVtMfJXHS6kb%252Fgrafana_dashboards.png%3Fgeneration%3D1540136069596798%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=171421a0&sv=1)

用户共享的Dashboard



---


Grafana中所有的Dashboard通过JSON进行共享，下载并且导入这些JSON文件，就可以直接使用这些已经定义好的Dashboard：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPMFlGDFIX7wuLhSHx9%252F-LPMFp7BMwAjVVS2GA8a%252Fnode_exporter_dashboard.png%3Fgeneration%3D1540136069618026%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=651886e&sv=1)

Host Stats Dashboard


