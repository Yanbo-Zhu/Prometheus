

# 1 例子1

https://www.yuque.com/ssebank/eoudn8/zwin11
## 1.1 添加数据源

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627135032703-7ed3481c-67a7-4107-a35a-0eef3ff54384.png)

## 1.2 创建监控图标

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627109501905-5371fde0-d29f-4775-b1e7-3291bf091e2c.png)

## 1.3 网卡测试数据

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627135238419-d964e5bb-6ff1-4c16-b21b-f31f74ef133f.png)

```
panel中操作
- 设置单位
- panel改名
- 曲线别名
- 曲线sort
- 曲线复制
- 曲线静默
- panel复制
- 设置告警线
```

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627109876287-d6d9deb2-84ee-4036-bb11-47c881c06fbd.png)

## 1.4 设置table

使用变量查询

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627135670675-40c8cc15-1228-49f1-a9b9-50b8c8134d11.png)

## 1.5 排序

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627110223971-7a5a905d-83b9-413d-976c-760351bb3479.png)

## 1.6 图形tag

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627110272612-796b387a-8567-4ff0-ba68-415aecb59807.png)

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627110324931-68ae81bc-034c-41dd-ae82-7de4ca261107.png)

  

## 1.7 创建分组

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627109812692-ff7b90ad-c086-4308-baaa-b2d923cee38c.png)![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627109812700-3dacf210-28f8-4f56-b211-7fd40e8e8d92.png)

## 1.8 变量设置

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627110573072-6005b06a-bc21-4b23-aa04-855f40872e71.png)

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627110597923-f0b48f99-894d-49a6-a4aa-8b0cbe49a7fc.png)

## 1.9 外链

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627110623294-c035a98b-2136-448e-a770-8417a3801d15.png)

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627110639781-74b83d7b-5012-4b48-a19b-d11e1e6ec4fa.png)

## 1.10 变量嵌套

```
> dashboard开屏时需要加载变量，这是导致他们慢的原因之一


> 导入dashboard商城中的node_exporter模板

- 地址 https://grafana.com/grafana/dashboards

- 两种导入模式
  - url导入
  - json文件导入
- https://grafana.com/grafana/dashboards/8919
```

![](https://cdn.nlark.com/yuque/0/2021/png/276796/1627136003116-4de82d55-00be-4a03-91f1-3ef2252a6635.png)





# 2 #

Grafanf 界面地址: ip+3000  
初始账号密码: admin, admin  
这里我依旧用图片加注释来进行讲解，想必这样会更容易理解吧。。。

https://www.cnblogs.com/xuwujing/p/14065740.html

## 2.1 主界面信息



![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121430270-1404124744.png)

## 2.2 创建仪表盘监控实现

1.添加数据源

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121430697-676694054.png)

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121431000-61348908.png)  


2.选择prometheus  
![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121431283-720064545.png)

3.点击创建仪表盘

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121431489-910717320.png)


4.点击创建

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121431700-633621399.png)

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121431960-94887268.png)

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121432168-928827487.png)

5.输入node_load1表示语句，填写相关信息，点击apply完成，并将名称保存为Test  
![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121432472-396027243.png)

6.点击搜索Test，点击就可以查看

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121432682-1202217606.png)

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121432836-1970813110.png)

## 2.3 使用第三方仪表盘监控实现

注:需提前添加好数据源。

1.点击左上角的加号，点击import

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121433062-1732099602.png)

**在线模式**  
地址:[https://grafana.com/grafana/dashboards](https://grafana.com/grafana/dashboards)

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121433339-1124422177.png)

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121433955-1237232905.png)

**离线模式**

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121434261-469773227.png)

**监控示例:**

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121434567-346275305.png)


# 3 

 **步骤二：配置数据源**

grafana支持多种数据源，可以在“type”的下拉框选项中看到，这里我们选择prometheus作为数据源。HTTP的访问方式选择proxy，URL填写grafana能访问到的地址。

`因为grafana和prometheus都在同一个k8s集群中，这里用svc地址`

 ![](https://img2018.cnblogs.com/blog/1354564/201907/1354564-20190709084949182-310813946.png)

点击“save and test”测试连接性。




**步骤三：创建面板**

 ![](https://img2018.cnblogs.com/blog/1354564/201907/1354564-20190709085045407-422325431.png)

有了数据源，接下来就是如何更好地展示数据，grafana支持多种类型的图表，如Graph、singlestat、Table等。可以组合出多种形式。这里先创建一个Demo，保存现有模板的快捷键是Ctrl + S

 ![](https://img2018.cnblogs.com/blog/1354564/201907/1354564-20190709085107146-751148920.png)

你的所有面板都可以在左上角的下拉框中找到：

![](https://img2018.cnblogs.com/blog/1354564/201907/1354564-20190709085125906-1200131088.png)

我们还可以导入现有的面板（Dashboard），大家贡献的模板地址：[https://grafana.com/dashboards](https://grafana.com/dashboards)

 ![](https://img2018.cnblogs.com/blog/1354564/201907/1354564-20190709085145744-756078647.png)

选择左上角的➕号，然后import，有两种形式的导入：

- URL直接导入：粘贴对应模板的url，前提是grafana必须能访问外网。
- 上传json文件：一般是迁移时使用，把现有的json导出，再导入

grafana的面板、图表有很多配置，接下来我们说几个常用的配置项