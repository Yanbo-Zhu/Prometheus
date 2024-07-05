
在前面的文章已经写了官方的几个exporter的使用了。 在实际使用环境中，我们可能需要收集一些自定义的数据， 这个时候我们一般是需要自己编写采集器的。


# 1 监听某个端口 

只需要一个py文件， 运行起来， 会监听在8000端口，访问127.0.0.1:8000端口。
```
from prometheus_client import Counter, Gauge, Summary, Histogram, start_http_server

# need install prometheus_client

if __name__ == '__main__':
    c = Counter('cc', 'A counter')
    c.inc()

    g = Gauge('gg', 'A gauge')
    g.set(17)

    s = Summary('ss', 'A summary', ['a', 'b'])
    s.labels('c', 'd').observe(17)

    h = Histogram('hh', 'A histogram')
    h.observe(.6)

    start_http_server(8000)
    import time

    while True:
        time.sleep(1)
```

![](https://img2018.cnblogs.com/blog/429277/201910/429277-20191011085736417-1150927557.png)


 其实一个导出器就已经写好了， 就是这么简单的，我们只需要在prometheus配置来采集对应的导出器就可以了。 不过我们的导出的数据都是没有实际意义了。

# 2 采集内存使用数据 


编写采集类代码
```
from prometheus_client.core import GaugeMetricFamily, REGISTRY
from prometheus_client import start_http_server
import psutil


class CustomMemoryUsaggeCollector():
    def format_metric_name(self):
        return 'custom_memory_'

    def collect(self):
        vm = psutil.virtual_memory()
        #sub_metric_list = ["free", "available", "buffers", "cached", "used", "total"]
        sub_metric_list = ["free", "available", "used", "total"]
        for sub_metric in sub_metric_list:
            gauge = GaugeMetricFamily(self.format_metric_name() + sub_metric, '')
            gauge.add_metric(labels=[], value=getattr(vm, sub_metric))
            yield gauge


if __name__ == "__main__":
    collector = CustomMemoryUsaggeCollector()
    REGISTRY.register(collector)
    start_http_server(8001)
    import time
    while True:
        time.sleep(1)
```


![](https://img2018.cnblogs.com/blog/429277/201910/429277-20191011145859280-526107045.png)


部署代码和集成prometheus

```
# 准备python3 环境 参考： https://virtualenvwrapper.readthedocs.io/en/latest/
yum install python36 -y

 pip3 install virtualenvwrapper
vim /usr/local/bin/virtualenvwrapper.sh 
# 文件最前面添加如下行
# Locate the global Python where virtualenvwrapper is installed.
VIRTUALENVWRAPPER_PYTHON="/usr/bin/python3"

# 文件生效
source /usr/local/bin/virtualenvwrapper.sh
# 配置workon
[root@node01 ~]# echo "export WORKON_HOME=~/Envs" >>~/.bashrc 
[root@node01 ~]# mkvirtualenv  custom_memory_exporter
(custom_memory_exporter) [root@node01 ~]# pip install prometheus_client psutil
yum install python36-devel

(custom_memory_exporter) [root@node01 ~]# chmod a+x custom_memory_exporter.py 
(custom_memory_exporter) [root@node01 ~]# ./custom_memory_exporter.py 
# 测试是否有结果数据
[root@node00 ~]# curl http://192.168.100.11:8001/  
  
prometheus.yml 加入如下片段  
  - job_name: "custom-memory-exporter"  
    static_configs:  
    - targets: ["192.168.100.11:8001"]  
  
[root@node00 prometheus]# systemctl restart prometheus   
[root@node00 prometheus]# systemctl status prometheu
```


![](https://img2018.cnblogs.com/blog/429277/201910/429277-20191011151728193-602210400.png)





