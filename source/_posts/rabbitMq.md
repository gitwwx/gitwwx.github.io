---
title: Window下安装RabbitMQ
date: 2025-12-25 12:10:00
tags:
  - RabbitMQ
---

### 安装步骤
#### 第一步：下载rabbitmq
官网下载地址: https://www.rabbitmq.com/docs/install-windows
![官网地址](/medias/images/rabbitmq1.png)

#### 第二步: 下载Erlang
RabbitMQ需要安装64位受支持的Erlang版本 才能在Windows上运行。
![下载页面](/medias/images/rabbitmq2.png)

#### 第三步：点击安装
##### 1.先安装Erlang
* 运行程序安装
* 配置环境变量

配置ERLANG_HOME环境变量，其值指向erlang的安装目录（安装时可选择）<br>
%ERLANG_HOME%\bin 加入到Path中

验证安装 cmd窗口 输入 
```
erl -version
```

##### 2.安装rabbitmq-server
注意，安装的路径都不要包含中文<br>
* 配置环境变量

新建RABBITMQ_SERVER,指向安装路径，增加rabbitmq变量至path，%RABBITMQ_SERVER%\sbin

验证是否安装成功 运行以下命令 
```
rabbitmqctl status
```

##### 3.报错解决

Error: unable to perform an operation on node 'rabbit@DESKTOP-63FTMEF'. Please see diagnostics information and suggestions below.

* 找到：C:\Windows\System32\config\systemprofile .erlang.cookie 这个文件和 C:\Users\{用户名} .erlang.cookie 两个的值不一样 设置一样即可
* 重启 RabbitMQ 服务（Windows + R -> services.msc -> 找到 RabbitMQ -> 右击重启）

##### 4.扩展
切换到安装目录，安装可视化插件：
```
rabbitmq-plugins.bat enable rabbitmq_management
```

浏览器中 http://localhost:15672
* 用户和密码输入：guest,guest 进入rabbitMQ管理控制台
