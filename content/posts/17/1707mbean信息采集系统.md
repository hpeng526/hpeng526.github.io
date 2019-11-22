---
title: mbean信息采集系统
date: 2017-07-25 23:13:51
tags: [spring boot Actuator, spring, jolokia, Telegraf, influxdb, Grafana, MBeans]
---

# mbean 信息采集系统

### spring boot Actuator

这个不多讲，主要是利用 `metrics` 采集时间序列（time-series）数据，他也可以在 spring mvc 上使用，所以很容易在我们现有的系统上集成。

<!-- more -->

### jolokia

[jolokia.org](https://jolokia.org/)

简单点说，就是把 MBeans 以 RESTful HTTP 端点的方式暴露（json数据）。这就十分方便了，以前我们看 mbean 信息就只能通过 jconsole 啊之类的程序查看，其内部的实现都是非常重的 RMI 协议。其实 jolokia 也提供了非常多的方式方法来让我们通过浏览器查看当前应用注册的 mbean 信息，但是，如果要聚合一堆应用查看，这就不行了。

### Telegraf

[influxdata/telegraf](https://github.com/influxdata/telegraf)

Telegraf 是一个用 Go 写的代理，用来收集，处理，聚合和写 metrics。 你可以把他类比 `filebeats`（Logstash）。这个工具呢，他有输入输出插件，输入插件可以读取jolokia数据, 输出插件可以把数据输出到 influxdb 上。

### influxdb

[influxdata](https://www.influxdata.com/)

influxdb是目前比较流行的时间序列(time-series)数据库。

#### 什么是时间序列数据库

什么是时间序列数据库，最简单的定义就是数据格式里包含Timestamp字段的数据。时间序列数据的更重要的一个属性是如何去查询它，包括数据的过滤，计算等等。

它有以下几大特点：

- schemaless(无结构)，可以是任意数量的列；
- min, max, sum, count, mean, median 一系列函数，方便统计；
- Native HTTP API, 内置http支持，使用http读写；
- Powerful Query Language 类似sql；

### Grafana

简单点，它是一个开源仪表盘工具，目前支持26种数据源。当然有 influxdb ，也支持 ES。把它类比 Kibana 就清楚它的作用了。

### 拼凑原理

讲这么多，他们是怎么拼起来的？

1. Spring Boot Actuator 可以与 jolokia 无缝集成，提供了官方的支持，具体的实现就是，通过它的 auto config （JolokiaAutoConfiguration.class），把 jolokia 的配置通过 actuator 的 endpoint 暴露出去。这样，我们就可以通过 URL 采集 web 应用的 mbean 信息了。

    ```java
    public class JolokiaMvcEndpoint extends AbstractNamedMvcEndpoint implements InitializingBean, ApplicationContextAware, ServletContextAware, DisposableBean {
        private final ServletWrappingController controller = new ServletWrappingController();

        public JolokiaMvcEndpoint() {
            super("jolokia", "/jolokia", true);
            // 这个 AgentServlet.class 是jolokia用来在应用中暴露自身 endpoint 使用的
            // spring boot actuator 将它注册到 spring mvc controller 中。
            // ServletWrappingController 将 Servlet 直接包装为一个 Controller
            this.controller.setServletClass(AgentServlet.class);
            this.controller.setServletName("jolokia");
        }
        /*..省略..*/
    }
    ```

2. Telegraf 通过配置的需要采集的服务器信息，自动进行采集处理，并保存数据在 influxdb 中

3. Grafana 连接到 influxdb， 在页面定制（influxdb 的查询语句）要展示的图标，进行数据的展示。

简单画个图，就是

```
+---------+   /jolokia               +----------+
|  App 1  +-------------+            |          |
+---------+             |  telegraf  |          |
                        +----------->+ InfluxDB |
+---------+             |            |          |
|  App 2  +-------------+            |          |
+---------+                          +------+---+
                                            ^
                                fetch data  |
       +-------------------------------+    |
       |                               |    |
       |                               |    |
       |            Grafana            +----+
       |                               |
       |                               |
       +-------------------------------+
```
