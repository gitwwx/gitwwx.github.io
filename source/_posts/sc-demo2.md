---
title: SpringCloud微服务-Nacos + MQ
date: 2025-12-26 12:10:00
tags:
  - Nacos
  - 微服务 
  - MQ
---


#### 一、开发环境与工具
* jdk8
* maven
* Intellij IDEA

#### 二、工程构建
##### 2.1 构建父子工程
###### 2.1.1 创建父工程
1.创建一个spring boot项目或者空项目，去掉src目录，只保留pom.xml
目录结构如图
   ![图片](/medias/images/sc1.png)
   ![图片](/medias/images/sc2.png)
2.完善父工程pom文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
   <modelVersion>4.0.0</modelVersion>
   <groupId>com.wwx</groupId>
   <artifactId>cloud_demo</artifactId>
   <version>1.0</version>
   <name>cloud_demo</name>
   <packaging>pom</packaging>
   <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>2.7.6</version>
      <relativePath/> <!-- lookup parent from repository -->
   </parent>
   <properties>
      <java.version>1.8</java.version>
      <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
      <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
      <spring-cloud.version>2021.0.7</spring-cloud.version> <!-- 兼容Spring Boot 2.7.x -->
      <spring-cloud-alibaba.version>2021.0.5.0</spring-cloud-alibaba.version>
   </properties>
   <dependencyManagement>
      <dependencies>
         <!-- Nacos 依赖版本-->
         <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
            <version>${spring-cloud-alibaba.version}</version>
         </dependency>
         <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
            <version>${spring-cloud-alibaba.version}</version>
         </dependency>
      </dependencies>
   </dependencyManagement>

   <build>
      <pluginManagement>
         <plugins>
            <!-- 编译插件 -->
            <plugin>
               <groupId>org.apache.maven.plugins</groupId>
               <artifactId>maven-compiler-plugin</artifactId>
               <version>3.10.1</version>
               <configuration>
                  <source>${maven.compiler.source}</source>
                  <target>${maven.compiler.target}</target>
               </configuration>
            </plugin>
            <!-- Spring Boot Maven 插件 -->
            <plugin>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
         </plugins>
      </pluginManagement>
   </build>

</project>

```
父POM使用dependencyManagement统一管理依赖版本<br>
子模块在dependencies中声明实际需要的依赖，可以省略版本号<br>
单模块项目可以不用dependencyManagement，直接在dependencies中声明
###### 2.1.2 创建子项目通用model
![图片](/medias/images/sc3.png)
其他模块创建方式类似，model也可以选择创建springboot项目再编辑pom文件即可<br>
项目结构
```
spring-cloud-demo/
├── cloud-common          # 公共模块
├── user-service          # 用户服务
├── order-service         # 订单服务
├── gateway-service       # API网关
└── nacos-config          # Nacos配置管理
```

##### 2.2 搭建Nacos服务发现中心
下载并启动Nacos Server，安装启动教程详见本博客另一篇文章
```
# 单机模式启动 linux
sh startup.sh -m standalone

# 进入nacos/bin目录下 window
./startup.cmd -m standalone
```

##### 2.3 所有微服务添加依赖
```xml
<dependencies>
   <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
   </dependency>
</dependencies>
```

##### 2.4 配置Nacos连接

```yaml
spring:
   config:
      import: nacos:${spring.application.name}   # 加载通用配置
   cloud:
      nacos:
         discovery:
            server-addr: 127.0.0.1:8848
            namespace: 0475ca81-cb6f-454e-9882-5972fee62210  # 命名空间隔离
            group: CLOUD_DEMO
```
bootstrap.yml 在 Spring Cloud 2020.x（及更高版本）中默认已被弃用，改为使用 application.yml + 显式导入配置的方式
* 直接在 application.yml 中配置 Nacos，但需通过 spring.config.import 显式导入 Nacos 配置
##### 2.5 集成消息队列

```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```
启动RabbitMQ 服务，安装启动教程详见本博客另一篇文章

##### 2.6 配置RabbitMQ

```yaml
spring:
   rabbitmq:
      host: localhost
      port: 5672
      username: guest
      password: guest
```
创建消息生产者(订单服务)：
```java
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class OrderController {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    @PostMapping("/orders")
    public String createOrder() {
        // 业务逻辑...
        OrderDTO orderDTO = new OrderDTO();
        orderDTO.setOrderId("test123");
        orderDTO.setCustomerId("test456");
        rabbitTemplate.convertAndSend("order.event.exchange", 
                                    "order.create", 
                                    orderDTO);
        return "Order created";
    }
}
```
创建消息消费者(用户服务)：
```java
import org.springframework.amqp.rabbit.annotation.Exchange;
import org.springframework.amqp.rabbit.annotation.Queue;
import org.springframework.amqp.rabbit.annotation.QueueBinding;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.core.Message;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;

@Component
public class OrderEventListener {

   @RabbitListener(bindings = @QueueBinding(
           value = @Queue(value = "user.order.queue", durable = "true"),
           exchange = @Exchange(value = "order.event.exchange", type = "topic"),
           key = "order.*"
   ))
   public void handleOrderEvent(OrderDTO order) {
      // 处理订单事件
      System.out.println(order.getOrderId());
   }

   /**
    * 两个 @RabbitListener 方法监听 同一个队列（user.order.queue），RabbitMQ 会：
    * 轮询分发消息（Round-Robin）给不同的消费者。
    * 不会重复消费同一条消息（即一条消息只会被一个监听器处理）。
    * @param order
    * @param routingKey
    * @param message
    */
   @RabbitListener(bindings = @QueueBinding(
           value = @Queue(value = "user.order.queue", durable = "true"),
           exchange = @Exchange(value = "order.event.exchange", type = "topic"),
           key = "order.cancel"
   ))
   public void handleOrderCancel(OrderDTO order, @Header("amqp_receivedRoutingKey") String routingKey, Message message) {
      // 处理订单取消事件
      String routingKey2 = message.getMessageProperties().getReceivedRoutingKey();
   }
}
```
##### 2.7 搭建API网关

```xml
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```
Eureka 时代的负载均衡
* Netflix Ribbon（旧版默认） 在早期的 Spring Cloud 版本（如 Hoxton 及之前），当使用 Eureka 作为注册中心时，spring-cloud-starter-netflix-ribbon 会自动被间接引入（例如通过 spring-cloud-starter-netflix-eureka-client）
```
示例依赖传递路径：
eureka-client → ribbon → ribbon-loadbalancer
```

Spring Cloud 2020 后的变化

* Ribbon 进入维护模式 Netflix 宣布 Ribbon 进入维护状态，Spring Cloud 逐步移除对 Ribbon 的默认依赖。
* Spring Cloud LoadBalancer（SCL） Spring 官方推出了新的负载均衡器 spring-cloud-loadbalancer，成为默认实现（从 2020.0.x 开始）。

Gateway 的负载均衡依赖

* lb://service-name 语法依赖负载均衡器 Gateway 的路由配置中 uri: lb://user-service 的 lb 前缀需要底层负载均衡器支持。
* Spring Cloud Gateway 的轻量化设计 Gateway 的 Starter 默认不传递负载均衡依赖（避免强绑定某一实现），需开发者显式选择。

```yaml
# 配置路由规则
server:
   port: 9090
spring:
   cloud:
      gateway:
         routes:
            - id: user-service
              uri: lb://user-service
              predicates:
                 - Path=/api/users/**
              filters:
                 - StripPrefix=1
```

##### 2.8 配置中心(Nacos Config)

```xml
<dependency>
   <groupId>com.alibaba.cloud</groupId>
   <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```
配置application.yml
```yaml
spring:
   application:
      name: nacos-config
   config:
      import: nacos:${spring.application.name}   # 加载通用配置
   profiles:
      active: dev
   cloud:
      nacos:
         config:
            server-addr: 127.0.0.1:8848
            file-extension: yaml
            namespace: 0475ca81-cb6f-454e-9882-5972fee62210
            group: CLOUD_DEMO
         # 服务发现（Discovery）设置
         discovery:
            server-addr: 127.0.0.1:8848
            namespace: 0475ca81-cb6f-454e-9882-5972fee62210
            group: CLOUD_DEMO  # 服务注册分组（关键！）
```
在Nacos控制台添加配置
```
Data ID: user-service
Group: CLOUD_DEMO
配置内容:
server:
  port: 8081
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/user_db
    username: root
    password: 123456
```
其他服务引入配置示例
```yaml
spring:
  profiles:
    active: dev
  application:
    name: user-service
  config:
    import: nacos:${spring.application.name}   # 加载通用配置
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: 0475ca81-cb6f-454e-9882-5972fee62210  # 命名空间隔离
        group: CLOUD_DEMO
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        group: CLOUD_DEMO
        namespace: 0475ca81-cb6f-454e-9882-5972fee62210
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
```

##### 2.9 服务调用(OpenFeign)
添加依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```
订单服务创建Feign客户端：
```java
@FeignClient(name = "user-service")
public interface UserClient {
    
    @GetMapping("/users/{id}")
    UserDTO getUserById(@PathVariable Long id);
}
```
在启动类添加注解：
```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

@EnableFeignClients
@SpringBootApplication
@EnableFeignClients
public class OrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderApplication.class, args);
    }
}
```


#### 三、测试与启动
##### 3.1 启动顺序
1. 启动Nacos Server
2. 启动RabbitMQ
3. 启动基础服务(按顺序)： 

3. * 配置中心(nacos-config)
3. * 网关服务(gateway-service)

4. 启动业务服务(可并行)：

4. * user-service
4. * order-service
##### 3.2 测试验证
  -  访问Nacos控制台(http://localhost:8848) 查看服务注册情况
  -  通过网关访问服务：GET http://localhost:9090/api/users/1
  -  测试消息队列：POST http://localhost:8083/orders
