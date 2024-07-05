
注意： 如果调试过程中有问题， 请查看/var/log/message信息，获取alertmanager发送邮件的错误信息。

# 1 总览

前上一小节已经讲过，在Alertmanager中路由负责对告警信息进行分组匹配，并将像告警接收器发送通知。告警接收器可以通过以下形式进行配置：
```
receivers:
  - <receiver> ...
```


每一个receiver具有一个全局唯一的名称，并且对应一个或者多个通知方式：
```
name: <string>
email_configs:
  [ - <email_config>, ... ]
hipchat_configs:
  [ - <hipchat_config>, ... ]
pagerduty_configs:
  [ - <pagerduty_config>, ... ]
pushover_configs:
  [ - <pushover_config>, ... ]
slack_configs:
  [ - <slack_config>, ... ]
opsgenie_configs:
  [ - <opsgenie_config>, ... ]
webhook_configs:
  [ - <webhook_config>, ... ]
victorops_configs:
  [ - <victorops_config>, ... ]
```

目前官方内置的第三方通知集成包括：邮件、 即时通讯软件（如Slack、Hipchat）、移动应用消息推送(如Pushover)和自动化运维工具（例如：Pagerduty、Opsgenie、Victorops）。Alertmanager的通知方式中还可以支持Webhook，通过这种方式开发者可以实现更多个性化的扩展支持。



# 2 集成邮件系统

邮箱应该是目前企业最常用的告警通知方式，Alertmanager内置了对SMTP协议的支持，因此对于企业用户而言，只需要一些基本的配置即可实现通过邮件的通知。

在Alertmanager使用邮箱通知，用户只需要定义好SMTP相关的配置，并且在receiver中定义接收方的邮件地址即可。在Alertmanager中我们可以直接在配置文件的global中定义全局的SMTP配置：

```
global:
  [ smtp_from: <tmpl_string> ]
  [ smtp_smarthost: <string> ]
  [ smtp_hello: <string> | default = "localhost" ]
  [ smtp_auth_username: <string> ]
  [ smtp_auth_password: <secret> ]
  [ smtp_auth_identity: <string> ]
  [ smtp_auth_secret: <secret> ]
  [ smtp_require_tls: <bool> | default = true ]
```

完成全局SMTP之后，我们只需要为receiver配置email_configs用于定义一组接收告警的邮箱地址即可，如下所示：

```
name: <string>
email_configs:
  [ - <email_config>, ... ]
```

每个email_config中定义相应的接收人邮箱地址，邮件通知模板等信息即可，当然如果当前接收人需要单独的SMTP配置，那直接在email_config中覆盖即可：

```
[ send_resolved: <boolean> | default = false ]
to: <tmpl_string>
[ html: <tmpl_string> | default = '{{ template "email.default.html" . }}' ]
[ headers: { <string>: <tmpl_string>, ... } ]
```

如果当前收件人需要接受告警恢复的通知的话，在email_config中定义`send_resolved`为true即可。

如果所有的邮件配置使用了相同的SMTP配置，则可以直接定义全局的SMTP配置。


## 2.1 Gmail邮箱为例

这里，以Gmail邮箱为例，我们定义了一个全局的SMTP配置，并且通过route将所有告警信息发送到default-receiver中:

```
global:
  smtp_smarthost: smtp.gmail.com:587
  smtp_from: <smtp mail from>
  smtp_auth_username: <usernae>
  smtp_auth_identity: <username>
  smtp_auth_password: <password>

route:
  group_by: ['alertname']
  receiver: 'default-receiver'

receivers:
  - name: default-receiver
    email_configs:
      - to: <mail to address>
        send_resolved: true
```

> 需要注意的是新的Google账号安全规则需要使用”应用专有密码“作为邮箱登录密码

这时如果手动拉高主机CPU使用率，使得监控样本数据满足告警触发条件。在SMTP配置正确的情况下，可以接收到如下的告警内容：

![](https://yunlzheng.gitbook.io/~gitbook/image?url=https%3A%2F%2F2416223964-files.gitbook.io%2F%7E%2Ffiles%2Fv0%2Fb%2Fgitbook-legacy-files%2Fo%2Fassets%252F-LBdoxo9EmQ0bJP2BuUi%252F-LPS9OCY3C6XBe9KTNE4%252F-LPS9R8NSvST8OiYzugr%252Fmail-alert-page.png%3Fgeneration%3D1540235057857854%26alt%3Dmedia&width=768&dpr=4&quality=100&sign=31ecb4d&sv=1)


## 2.2 qq邮箱

```yaml
[root@node00 alertmanager]# cat alertmanager.yml
global:
  resolve_timeout: 5m

  ###################################
  smtp_auth_username: "1072892917@qq.com"
  smtp_auth_password: "这是你的QQ邮箱授权码而不是密码，切记切记，具体授权码获取看后面的本文末尾介绍有"
  #smtp_auth_secret: ""
  smtp_require_tls: false
  smtp_smarthost: "smtp.qq.com:465"
  smtp_from: "1072892917@qq.com"
  ####################################


route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'email-zhaojiedi'
  
receivers:
- name: 'email-zhaojiedi'
  email_configs:
  - send_resolved: true
    to: 1072892917@qq.com
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```


使用如下命令触发报警
```
[root@node01 ~]# dd if=/dev/zero of=bigfile bs=1M count=40000
```




# 3 调试过程中常用错误


```
问题1：

　　 does not advertise the STARTTLS extension： 

解决方案：

　　smtp_require_tls: false即可。

问题2：

　　email.loginAuth auth: 535 Error

解决方案：

 　　smtp_auth_password: 这个配置项设置为授权码，而不是QQ邮箱登陆，详细获取授权码参考地址： https://zhidao.baidu.com/question/878811848141402332.html

问题3：

　　一切配置正确，就是发不出去。

解决方案：

　　查看是否使用了25端口，默认云厂商会禁用25端口， 可以向云厂商申请解封25端口，或者使用465端口。

问题4：

　　报警消息能发送，但是报警恢复消息收不到。

解决方案：

　　缺少 send_resolved: true 配置文件， 请确保对应email_config配置文件，有此属性。
```
