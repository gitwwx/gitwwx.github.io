---
title: Nacos Server 下载与启动教程
date: 2025-12-25 12:10:00
tags:
  - Nacos
---

### 安装步骤
#### 第一步：下载Nacos Server
官网下载地址: 🔗 https://github.com/alibaba/nacos
选择最新稳定版本（注意jdk版本适配），下载对应系统的压缩包：

* Windows：nacos-server-x.x.x.zip
* Linux/macOS：nacos-server-x.x.x.tar.gz

#### 第二步: 解压文件
```
# Linux/macOS
tar -zxvf nacos-server-x.x.x.tar.gz

# Windows
直接右键解压
```

### 启动 Nacos Server
##### 1.单机模式启动（开发环境推荐）
Linux/macOS
```
cd nacos/bin
sh startup.sh -m standalone  # 添加 standalone 参数表示单机模式
```
Windows
```
cd nacos\bin
./startup.cmd -m standalone
```
##### 2.集群模式启动（生产环境）
修改配置文件 conf/cluster.conf，添加集群节点IP：
```
   192.168.1.101:8848
   192.168.1.102:8848
   192.168.1.103:8848
```
启动每个节点（无需 standalone 参数）
```
   sh startup.sh  # Linux/macOS
   startup.cmd    # Windows
```
### 验证启动
访问控制台
浏览器打开：
🔗 http://localhost:8848/nacos

* 默认账号：nacos
* 默认密码：nacos

查看日志
```
tail -f nacos/logs/start.out  # Linux/macOS
# Windows 直接查看 nacos\logs\start.out 文件
```
### 常见问题解决
#### 1，端口冲突
修改 conf/application.properties 中的端口：
```properties
  server.port=8848  # 默认端口，可改为其他
```
#### 2.启动失败（内存不足）
调整 JVM 参数（编辑 bin/startup.sh 或 startup.cmd）：
```
  JAVA_OPT="${JAVA_OPT} -Xms512m -Xmx512m"  # 根据机器配置调整
```
#### 3.外网访问限制
修改 conf/application.properties：
```properties
  nacos.core.auth.enabled=false  # 关闭鉴权（开发环境）
  nacos.inetutils.ip-address=你的服务器IP  # 绑定外网IP
```
### 线上环境建议
数据库持久化（默认使用内嵌 Derby，生产需切 MySQL） 
* 修改 conf/application.properties：
```properties
   spring.datasource.platform=mysql
   db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?useSSL=false
   db.user=root
   db.password=你的密码
```

### 关闭Nacos Server
```
sh shutdown.sh  # Linux/macOS
shutdown.cmd    # Windows
```