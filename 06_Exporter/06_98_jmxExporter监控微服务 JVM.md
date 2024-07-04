
https://www.cnblogs.com/JetpropelledSnake/p/16397810.html

本文实现微服务JVM监控的方法为，使用volume HostPath挂载的JMX Exporter的方式在容器内以in-process的方式实现对微服务的JVM监控。
主要坑点为HostPath挂载JMX Exporter javaagent导致微服务无法启动。


















