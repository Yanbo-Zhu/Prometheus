

由于 Prometheus 本身没有提供管理配置的 API 接口（尤其是管理监控目标和管理警报规则），也没有提供好用的多实例管理手段，因此这一块往往要自己写一些代码或脚本。

什么是 Operator？Operator = Controller + CRD。假如你不了解什么是 Controller 和 CRD，可以看一个 Kubernetes 本身的例子：我们提交一个 Deployment 对象来声明期望状态，比如 3 个副本；而 Kubernetes 的 Controller 会不断地干活（跑控制循环）来达成期望状态，比如看到只有 2 个副本就创建一个，看到有 4 个副本了就删除一个。
在这里，Deployment 是 Kubernetes 本身的 API 对象。

那假如我们想自己设计一些 API 对象来完成需求呢？Kubernetes 本身提供了 CRD(Custom Resource Definition)，允许我们定义新的 API 对象。但在定义完之后，Kubernetes 本身当然不可能知道这些 API 对象的期望状态该如何到达。这时，我们就要写对应的 Controller 去实现这个逻辑。而这种自定义 API 对象 + 自己写 Controller 去解决问题的模式，就是 Operator Pattern。


什么
