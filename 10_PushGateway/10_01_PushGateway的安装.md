
# 1 1，文件准备

将下载好的pushgateway文件解压

输入  
tar -zxvf pushgateway-1.2.0.linux-amd64.tar.gz  
然后移动到/opt/prometheus文件夹里面，没有该文件夹则创建

# 2 2，启动

root用户下启动

输入:

```bash
nohup ./pushgateway   >/dev/null   2>&1 &
```

启动成功之后，在浏览器上输入 ip+9091可以查看相关信息

![在这里插入图片描述](https://img2020.cnblogs.com/blog/1138196/202104/1138196-20210405121426839-1247309182.png)


