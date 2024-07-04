
grafana在v4.0版本开始引入了报警功能。

还是以Node节点数为例，我们配置一条规则：

当可用节点数小于3时，报警给demo@126.com

![](https://img2018.cnblogs.com/blog/1354564/201907/1354564-20190709085651185-2091381241.png)

alert支持avg、sum等表达式，不过持续时间依赖数据本身的采集频率。需要多测试一下。



# 1 Notifications

Notifications：配置报警的收件组和详细内容。而报警收件人的配置在专门的Alerting页面上

![](https://img2018.cnblogs.com/blog/1354564/201907/1354564-20190709085710304-1606416121.png)



# 2 Alert Rules

Alert Rules：已经配置的报警规则，并展示其触发状态。






# 3 报警邮件的样式


 ![](https://img2018.cnblogs.com/blog/1354564/201907/1354564-20190709085726033-289410620.png)




# 4 **模板变量报警**

以上的报警配置方式只适合没有变量传入的图表，如果遇到上边提到的选择node，传入变量的图表，就没办法支持了。

 ![](https://img2018.cnblogs.com/blog/1354564/201907/1354564-20190709085739360-58828081.png)

相关issue：[https://github.com/grafana/gr...](https://github.com/grafana/grafana/issues/9334)

官方对这个功能解释了一堆，最新版本仍然没有支持。借用issue的一句话哈哈哈

It would be grate to add templates support in alerts. Otherwise the feature looks useless a bit.

本文为容器监控实践系列文章，完整内容见：[container-monitor-book](https://yasongxu.gitbook.io/container-monitor/)


