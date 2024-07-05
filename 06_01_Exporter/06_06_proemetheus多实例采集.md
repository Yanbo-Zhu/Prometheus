https://www.yuque.com/ssebank/eoudn8/yeigi4


# 1 采集模式 

- pull模型
- push模型


> exporter分离 
- 单实例采集：必须本机部署的exporter
- 多实例采集：可以远端部署的exporter


> target配置模式
- 静态配置模式
- 动态服务发现模式

# 2 exporter 分类

![](image/Pasted%20image%2020240703224216.png)



# 3 proemetheus exporter 多实例采集说明

**为什么有多实例采集**
**·要么无法在目标服务器上运行程序，例如说网络设备的SNMP**
**·要么我们对距离（例如网站从网络外部特定点到站点的延迟和可访问性）特别感兴趣，这是常见的黑盒导出器的·用例。**
**·或者说部署agent代价比较大**

**多实例采集模式特点**
**exporter将通过网络协议获取目标的指标。**
**exporter不必在获取度量标准的计算机上运行。**
**Prometheus GET请求的参数作为 exporter获取目标和查询配置字符串**
**然后，exporter在收到Prometheus的GET请求之后开始抓取**
**exporter可以采集多个目标**


blackbox_exporter 需要传入target 和 module 参数，采用下列方式加入的采集池中 

```
  - job_name: 'blackbox-http'
    # metrics的path 注意不都是/metrics
    metrics_path: /probe
    # 传入的参数
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
      target: [prometheus.io,www.baidu.com,172.20.70.205:3000]
    static_configs:
      - targets:
        - 172.20.70.205:9115 
```


```
- 此方案的缺点
  - 实际采集目标位于参数配置中，这非常不寻常，以后很难理解。
  - 该instance标签显示的是`blackbox_exporter`的地址，从技术上讲是真实的，但不是我们感兴趣的内容。
  - 我们看不到我们探查了哪个URL。这是不切实际的，并且如果我们探查多个URL，也会将不同的指标混合到一个指标中。

- 解决方案说明：relabeling
  - 所有以__开头的标签在采集完成后都会被drop调。大多数内部标签以开头__
  - 可以设置一个内部标签形如`__param_<name>` ，代表设置URL参数 name=value 
  - 有一个内部标签__address__，在static_configs时由targets设置，其值是抓取请求的主机名。默认情况下，它被赋值给instance 标签，代表采集来源。
```


