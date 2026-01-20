---
title: JMeter安装使用教程
date: 2025-12-30 12:10:00
tags:
  - JMeter
---

#### 一、JMeter基础介绍
##### Apache JMeter 是一款开源的性能测试工具，主要用于：
* Web 应用压力测试
* API 接口性能测试
* 数据库负载测试
* 消息队列性能测试
#### 二、安装与配置
##### 1.下载安装
* 官网下载：https://jmeter.apache.org/download_jmeter.cgi
* 选择二进制包（如：apache-jmeter-5.6.3.zip）
* 解压到本地目录
##### 2.环境配置
JMeter是基于Java语言开发的，需安装jdk
##### 3.启动JMeter
```
# Windows
jmeter.bat

# Linux/Mac
jmeter.sh
```
#### 三、创建第一个测试计划（用户服务接口测试）
##### 1.创建测试计划
右键点击 "Test Plan" -> Add -> Threads (Users) -> Thread Group
* 设置参数：
* * Number of Threads (users): 100（并发用户数）
* * Ramp-up period (seconds): 10（10秒内启动所有用户）
* * Loop Count: 50（每个用户执行50次）

##### 2.添加 HTTP 请求
1. 右键 Thread Group -> Add -> Sampler -> HTTP Request
2. * 配置 HTTP 请求：
2. * Protocol: http
2. * Server Name or IP: localhost（网关地址）
2. * Port Number: 9090（网关端口）
2. * Method: GET
2. * Path: /api/users/${userId}（使用变量）

##### 3.添加随机变量
1. 右键 Thread Group -> Add -> Config Element -> Random Variable
2. * 配置：
2. * Variable Name: userId
2. * Minimum Value: 1
2. * Maximum Value: 1000

##### 4.添加结果监听器
1. 右键 Thread Group -> Add -> Listener -> View Results Tree
2. 右键 Thread Group -> Add -> Listener -> Summary Report
3. 右键 Thread Group -> Add -> Listener -> Response Time Graph

保存测试计划

* File -> Save As -> user_service_test.jmx

#### 四.创建订单服务测试计划
##### 1.添加第二个HTTP请求
右键 Thread Group -> Add -> Sampler -> HTTP Request

配置：
* Method: POST
* Path: /orders
* Body Data:
```json
     {
          "customerId": "${userId}",
          "productId": "${productId}"
     }
```
##### 2.添加HTTP头管理器
右键 HTTP Request -> Add -> Config Element -> HTTP Header Manager

添加头：
* Name: Content-Type
* Value: application/json

#### 五、高级配置技巧
##### 1.参数化测试（csv数据）
创建 CSV 文件 test_data.csv：
```
   userId,productId
   1,P100
   2,P200
   ...
```
右键 Thread Group -> Add -> Config Element -> CSV Data Set Config
* Filename: 文件路径
* Variable Names: userId,productId
* Delimiter: ,

##### 2.使用正则表达式提取数据
在第一个请求后添加 Post Processor：
* 右键 HTTP Request -> Add -> Post Processors -> Regular Expression Extractor

配置：
* Reference Name: extractedUserId
* Regular Expression: "id":(\d+)
* Template: 1

##### 3.添加断言验证
右键 HTTP Request -> Add -> Assertions -> Response Assertion

配置：
* Field to Test: Response Code
* Pattern Matching Rules: Equals
* Patterns: 200

##### 4.添加定时器（模拟真实用户行为）
右键 Thread Group -> Add -> Timer -> Gaussian Random Timer
* Deviation: 300 ms
* Constant Delay: 500 ms

#### 六、执行测试与结果分析
##### 1.执行测试
* 点击工具栏绿色箭头按钮

<div style="border: 1px solid #ddd; padding: 10px; border-radius: 5px; background: #f9f9f9;">
💡 提示: 使用 GUI 模式进行负载测试需要增加Java堆以满足测试要求<br>
修改jmeter.bat文件中当前环境变量HEAP="-Xms1g -Xmx1g -XX:MaxMetaspaceSize=256m"
</div>

* 或命令行执行：
```
  jmeter -n -t service_test.jmx -l results.jtl
  # -n: 非GUI模式 -t: 测试计划文件 -l: 结果文件
```
##### 2.结果分析
1.Summary Report:
* 查看吞吐量(Throughput)、平均响应时间(Average)
* 错误率(Error %)

2.Response Time Graph:
* 查看响应时间分布

3.聚合报告生成:
* 生成HTML报告
```
   jmeter -g results.jtl -o report/
```

#### 七、微服务专用测试方案
##### 1.网关压力测试配置
```yaml
HTTP Request 配置：
- Server: localhost
- Port: 9090
- Path: 
  - /user-service/users/${userId} (GET)
  - /order-service/orders (POST)
```

##### 2.分布式测试（多机负载）
1.控制机配置：
```properties
   # jmeter.properties
   remote_hosts=192.168.1.145,192.168.1.174
```
2.执行机配置：
```
jmeter-server.bat # Windows
jmeter-server     # Linux/Mac
```
3.配置rmi
```
# 控制机执行命令，将生成的 rmi_keystore.jks 文件复制到所有执行机的 JMeter bin 目录
keytool -genkey -alias rmi -keyalg RSA -keystore rmi_keystore.jks -validity 365 -keysize 2048 -storepass changeit -keypass changeit
```
4.控制机执行：
```
jmeter -n -t service_test.jmx -R 192.168.1.145,192.168.1.174 -l result2.jtl
```
##### 3.监控集成（Grafana）
1.添加 Backend Listener:
* 右键 Thread Group -> Add -> Listener -> Backend Listener

2.配置：
* Backend Listener implementation: GrafanaInfluxDBBackendListenerClient
* influxdbUrl: http://localhost:8086
* application: user-service

#### 八、常见问题解决
##### 1.内存溢出问题
修改 jmeter.bat (Windows) 或 jmeter (Linux)：
```
# 找到 HEAP 设置
set HEAP=-Xms4g -Xmx8g
```
##### 2.高并发下Socker错误
在 HTTP Request 中：
* Implementation: Java
* 勾选 "Use KeepAlive"

##### 3.结果文件过大
```
# 只保存错误信息
jmeter -j test.log -l results.jtl -Jjmeter.save.saveservice.assertion_results_failure_message=false
```
#### 九、完整测试计划示例
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0" jmeter="5.6.3">
    <hashTree>
        <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="Test Plan">
            <stringProp name="TestPlan.comments">Spring Cloud 压力测试</stringProp>
            <elementProp name="TestPlan.user_defined_variables" elementType="Arguments" guiclass="ArgumentsPanel" testclass="Arguments" testname="User Defined Variables">
                <collectionProp name="Arguments.arguments"/>
            </elementProp>
        </TestPlan>
        <hashTree>
            <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Thread Group">
                <stringProp name="TestPlan.comments">测试计划</stringProp>
                <intProp name="ThreadGroup.num_threads">100</intProp>
                <intProp name="ThreadGroup.ramp_time">10</intProp>
                <boolProp name="ThreadGroup.same_user_on_next_iteration">true</boolProp>
                <stringProp name="ThreadGroup.on_sample_error">continue</stringProp>
                <elementProp name="ThreadGroup.main_controller" elementType="LoopController" guiclass="LoopControlPanel" testclass="LoopController" testname="Loop Controller">
                    <stringProp name="LoopController.loops">10</stringProp>
                    <boolProp name="LoopController.continue_forever">false</boolProp>
                </elementProp>
            </ThreadGroup>
            <hashTree>
                <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="HTTP Request">
                    <stringProp name="TestPlan.comments">用户服务接口</stringProp>
                    <stringProp name="HTTPSampler.domain">localhost</stringProp>
                    <stringProp name="HTTPSampler.port">9090</stringProp>
                    <stringProp name="HTTPSampler.protocol">http</stringProp>
                    <stringProp name="HTTPSampler.path">/api/users/${userId}</stringProp>
                    <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
                    <stringProp name="HTTPSampler.method">GET</stringProp>
                    <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
                    <boolProp name="HTTPSampler.postBodyRaw">false</boolProp>
                    <elementProp name="HTTPsampler.Arguments" elementType="Arguments" guiclass="HTTPArgumentsPanel" testclass="Arguments" testname="User Defined Variables">
                        <collectionProp name="Arguments.arguments"/>
                    </elementProp>
                </HTTPSamplerProxy>
                <hashTree>
                    <RegexExtractor guiclass="RegexExtractorGui" testclass="RegexExtractor" testname="Regular Expression Extractor">
                        <stringProp name="RegexExtractor.useHeaders">false</stringProp>
                        <stringProp name="RegexExtractor.refname">extractedUserId</stringProp>
                        <stringProp name="RegexExtractor.regex">&quot;id&quot;:(\d+)</stringProp>
                        <stringProp name="RegexExtractor.template">1</stringProp>
                        <stringProp name="RegexExtractor.default"></stringProp>
                        <boolProp name="RegexExtractor.default_empty_value">false</boolProp>
                        <stringProp name="RegexExtractor.match_number"></stringProp>
                    </RegexExtractor>
                    <hashTree/>
                    <ResponseAssertion guiclass="AssertionGui" testclass="ResponseAssertion" testname="Response Assertion">
                        <collectionProp name="Asserion.test_strings">
                            <stringProp name="49586">200</stringProp>
                        </collectionProp>
                        <stringProp name="Assertion.custom_message"></stringProp>
                        <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
                        <boolProp name="Assertion.assume_success">false</boolProp>
                        <intProp name="Assertion.test_type">8</intProp>
                    </ResponseAssertion>
                    <hashTree/>
                </hashTree>
                <RandomVariableConfig guiclass="TestBeanGUI" testclass="RandomVariableConfig" testname="userId">
                    <stringProp name="maximumValue">1000</stringProp>
                    <stringProp name="minimumValue">1</stringProp>
                    <stringProp name="outputFormat"></stringProp>
                    <boolProp name="perThread">false</boolProp>
                    <stringProp name="randomSeed"></stringProp>
                    <stringProp name="variableName">userId</stringProp>
                    <stringProp name="TestPlan.comments">随机变量</stringProp>
                </RandomVariableConfig>
                <hashTree/>
                <ResultCollector guiclass="ViewResultsFullVisualizer" testclass="ResultCollector" testname="View Results Tree" enabled="true">
                    <boolProp name="ResultCollector.error_logging">false</boolProp>
                    <objProp>
                        <name>saveConfig</name>
                        <value class="SampleSaveConfiguration">
                            <time>true</time>
                            <latency>true</latency>
                            <timestamp>true</timestamp>
                            <success>true</success>
                            <label>true</label>
                            <code>true</code>
                            <message>true</message>
                            <threadName>true</threadName>
                            <dataType>true</dataType>
                            <encoding>false</encoding>
                            <assertions>true</assertions>
                            <subresults>true</subresults>
                            <responseData>false</responseData>
                            <samplerData>false</samplerData>
                            <xml>false</xml>
                            <fieldNames>true</fieldNames>
                            <responseHeaders>false</responseHeaders>
                            <requestHeaders>false</requestHeaders>
                            <responseDataOnError>false</responseDataOnError>
                            <saveAssertionResultsFailureMessage>true</saveAssertionResultsFailureMessage>
                            <assertionsResultsToSave>0</assertionsResultsToSave>
                            <bytes>true</bytes>
                            <sentBytes>true</sentBytes>
                            <url>true</url>
                            <threadCounts>true</threadCounts>
                            <idleTime>true</idleTime>
                            <connectTime>true</connectTime>
                        </value>
                    </objProp>
                    <stringProp name="filename"></stringProp>
                </ResultCollector>
                <hashTree/>
                <ResultCollector guiclass="SummaryReport" testclass="ResultCollector" testname="Summary Report" enabled="true">
                    <boolProp name="ResultCollector.error_logging">false</boolProp>
                    <objProp>
                        <name>saveConfig</name>
                        <value class="SampleSaveConfiguration">
                            <time>true</time>
                            <latency>true</latency>
                            <timestamp>true</timestamp>
                            <success>true</success>
                            <label>true</label>
                            <code>true</code>
                            <message>true</message>
                            <threadName>true</threadName>
                            <dataType>true</dataType>
                            <encoding>false</encoding>
                            <assertions>true</assertions>
                            <subresults>true</subresults>
                            <responseData>false</responseData>
                            <samplerData>false</samplerData>
                            <xml>false</xml>
                            <fieldNames>true</fieldNames>
                            <responseHeaders>false</responseHeaders>
                            <requestHeaders>false</requestHeaders>
                            <responseDataOnError>false</responseDataOnError>
                            <saveAssertionResultsFailureMessage>true</saveAssertionResultsFailureMessage>
                            <assertionsResultsToSave>0</assertionsResultsToSave>
                            <bytes>true</bytes>
                            <sentBytes>true</sentBytes>
                            <url>true</url>
                            <threadCounts>true</threadCounts>
                            <idleTime>true</idleTime>
                            <connectTime>true</connectTime>
                        </value>
                    </objProp>
                    <stringProp name="filename"></stringProp>
                </ResultCollector>
                <hashTree/>
                <ResultCollector guiclass="RespTimeGraphVisualizer" testclass="ResultCollector" testname="Response Time Graph" enabled="true">
                    <boolProp name="ResultCollector.error_logging">false</boolProp>
                    <objProp>
                        <name>saveConfig</name>
                        <value class="SampleSaveConfiguration">
                            <time>true</time>
                            <latency>true</latency>
                            <timestamp>true</timestamp>
                            <success>true</success>
                            <label>true</label>
                            <code>true</code>
                            <message>true</message>
                            <threadName>true</threadName>
                            <dataType>true</dataType>
                            <encoding>false</encoding>
                            <assertions>true</assertions>
                            <subresults>true</subresults>
                            <responseData>false</responseData>
                            <samplerData>false</samplerData>
                            <xml>false</xml>
                            <fieldNames>true</fieldNames>
                            <responseHeaders>false</responseHeaders>
                            <requestHeaders>false</requestHeaders>
                            <responseDataOnError>false</responseDataOnError>
                            <saveAssertionResultsFailureMessage>true</saveAssertionResultsFailureMessage>
                            <assertionsResultsToSave>0</assertionsResultsToSave>
                            <bytes>true</bytes>
                            <sentBytes>true</sentBytes>
                            <url>true</url>
                            <threadCounts>true</threadCounts>
                            <idleTime>true</idleTime>
                            <connectTime>true</connectTime>
                        </value>
                    </objProp>
                    <stringProp name="filename"></stringProp>
                </ResultCollector>
                <hashTree/>
                <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="HTTP Request">
                    <stringProp name="TestPlan.comments">订单服务接口</stringProp>
                    <stringProp name="HTTPSampler.domain">localhost</stringProp>
                    <stringProp name="HTTPSampler.port">8083</stringProp>
                    <stringProp name="HTTPSampler.protocol">http</stringProp>
                    <stringProp name="HTTPSampler.path">/orders</stringProp>
                    <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
                    <stringProp name="HTTPSampler.method">POST</stringProp>
                    <boolProp name="HTTPSampler.use_keepalive">true</boolProp>
                    <boolProp name="HTTPSampler.postBodyRaw">true</boolProp>
                    <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
                        <collectionProp name="Arguments.arguments">
                            <elementProp name="" elementType="HTTPArgument">
                                <boolProp name="HTTPArgument.always_encode">false</boolProp>
                                <stringProp name="Argument.value">{&#xd;
                                    &quot;customerId&quot;: &quot;${userId}&quot;,&#xd;
                                    &quot;productId&quot;: &quot;${productId}&quot;&#xd;
                                    }</stringProp>
                                <stringProp name="Argument.metadata">=</stringProp>
                            </elementProp>
                        </collectionProp>
                    </elementProp>
                </HTTPSamplerProxy>
                <hashTree>
                    <HeaderManager guiclass="HeaderPanel" testclass="HeaderManager" testname="HTTP Header Manager">
                        <collectionProp name="HeaderManager.headers">
                            <elementProp name="" elementType="Header">
                                <stringProp name="Header.name">Content-Type</stringProp>
                                <stringProp name="Header.value">application/json</stringProp>
                            </elementProp>
                        </collectionProp>
                    </HeaderManager>
                    <hashTree/>
                </hashTree>
                <CSVDataSet guiclass="TestBeanGUI" testclass="CSVDataSet" testname="CSV Data Set Config">
                    <stringProp name="delimiter">,</stringProp>
                    <stringProp name="fileEncoding"></stringProp>
                    <stringProp name="filename">D:/backRepo/demo1/test_data.csv</stringProp>
                    <boolProp name="ignoreFirstLine">false</boolProp>
                    <boolProp name="quotedData">false</boolProp>
                    <boolProp name="recycle">true</boolProp>
                    <stringProp name="shareMode">shareMode.all</stringProp>
                    <boolProp name="stopThread">false</boolProp>
                    <stringProp name="variableNames">userId,productId</stringProp>
                    <stringProp name="TestPlan.comments">订单服务测试数据</stringProp>
                </CSVDataSet>
                <hashTree/>
                <GaussianRandomTimer guiclass="GaussianRandomTimerGui" testclass="GaussianRandomTimer" testname="Gaussian Random Timer">
                    <stringProp name="ConstantTimer.delay">300</stringProp>
                    <stringProp name="RandomTimer.range">200.0</stringProp>
                </GaussianRandomTimer>
                <hashTree/>
                <ResultCollector guiclass="StatVisualizer" testclass="ResultCollector" testname="Aggregate Report" enabled="true">
                    <boolProp name="ResultCollector.error_logging">false</boolProp>
                    <objProp>
                        <name>saveConfig</name>
                        <value class="SampleSaveConfiguration">
                            <time>true</time>
                            <latency>true</latency>
                            <timestamp>true</timestamp>
                            <success>true</success>
                            <label>true</label>
                            <code>true</code>
                            <message>true</message>
                            <threadName>true</threadName>
                            <dataType>true</dataType>
                            <encoding>false</encoding>
                            <assertions>true</assertions>
                            <subresults>true</subresults>
                            <responseData>false</responseData>
                            <samplerData>false</samplerData>
                            <xml>false</xml>
                            <fieldNames>true</fieldNames>
                            <responseHeaders>false</responseHeaders>
                            <requestHeaders>false</requestHeaders>
                            <responseDataOnError>false</responseDataOnError>
                            <saveAssertionResultsFailureMessage>true</saveAssertionResultsFailureMessage>
                            <assertionsResultsToSave>0</assertionsResultsToSave>
                            <bytes>true</bytes>
                            <sentBytes>true</sentBytes>
                            <url>true</url>
                            <threadCounts>true</threadCounts>
                            <idleTime>true</idleTime>
                            <connectTime>true</connectTime>
                        </value>
                    </objProp>
                    <stringProp name="filename"></stringProp>
                    <boolProp name="useGroupName">true</boolProp>
                </ResultCollector>
                <hashTree/>
            </hashTree>
        </hashTree>
    </hashTree>
</jmeterTestPlan>

```
#### 十、最佳实践建议
##### 1.渐进式测试
* 从 10 用户开始，逐步增加并发量
* 每次增加后观察系统表现

##### 2.关键监控指标
```
# 使用 JMeter 插件管理器安装额外监控
https://jmeter-plugins.org/
```
* Transactions per Second (TPS)
* 95th Percentile Response Time
* Error Rate

##### 3.测试报告关键点：
* 确定系统瓶颈（CPU/内存/数据库/网络）
* 识别最佳并发用户数

