---
title: '实习总结：基于Prometheus + Grafana + JsonDatasource 搭建云资源监控系统'
tags:
  - 实习总结
  - 运维
  - 集群监控
  - GO语言
categories: 实习总结
abbrlink: 23692
date: 2020-06-15 18:20:00
abstract: 为了提升公司AI平台计算资源的利用效率，需要实现AI平台计算资源、应用数据的实时统计，故提出基于Prometheus+Grafana搭建资源监控平台。由于公司云资源分布在不同集群，需引入多个Prometheus DataSource，而Grafana对于多DataSource数据整合分析不具有很好的原生支持。所以使用Json Datasource插件，自行建立基于GO+Gin框架的后端，调用Prometheus Http接口，对不同集群数据进行整合计算，提供Http接口给Grafana平台调用以供展示。本文主要总结在实习过程中使用的上述组件的部分功能，并非对相关技术全面概览。
---

　　为了提升公司AI平台计算资源的利用效率，需要实现AI平台计算资源、应用数据的实时统计，故提出基于Prometheus+Grafana搭建资源监控平台。由于公司云资源分布在不同集群，需引入多个Prometheus DataSource，而Grafana对于多DataSource数据整合分析不具有很好的原生支持。所以使用Json Datasource插件，自行建立基于GO+Gin框架的后端，调用Prometheus Http接口，对不同集群数据进行整合计算，提供Http接口给Grafana平台调用以供展示。

　　以下主要总结在实习过程中使用的上述组件的部分功能，并非对相关技术全面概览。

---

# Prometheus
　　Prometheus 是一款基于**时序数据库**的开源监控告警系统，目前已经广泛用于 Kubernetes 集群的监控系统中。Prometheus 的开发者和用户社区非常活跃，它现在是一个独立的开源项目，可以独立于任何公司进行维护。

**主要特性**
- 多维度数据模型
- 方便的部署和维护
- 灵活的数据采集
- 强大的查询语言

　　Prometheus 数据采集方式也非常灵活。要采集目标的监控数据，首先需要在目标处安装数据采集组件，这被称之为 Exporter，它会在目标处收集监控数据，并暴露出一个 HTTP 接口供 Prometheus 查询。

　　Prometheus Server 直接从监控目标中或者间接通过推送网关来拉取监控指标，它在本地存储所有抓取到的样本数据，并对此数据执行一系列规则，以汇总和记录现有数据的新时间序列或生成告警。可以通过 Grafana 或者其他工具来实现监控数据的可视化。

## Prometheus 架构
![](/images/fig11.jpg "Prometheus 架构")


## 数据模型
　　Prometheus 所有采集的监控数据均以指标（metric）的形式保存在内置的时间序列数据库当中：属于同一指标名称，同一标签集合的、有时间戳标记的数据流。除了存储的时间序列，Prometheus 还可以根据查询请求产生临时的、衍生的时间序列作为返回结果。

### 样本
　　在时间序列中的每一个点称为一个样本（sample),由三部分组成：
- 指标（metric）：指标名称和描述当前样本特征的 labelsets
- 时间戳（timestamp）：一个精确到毫秒的时间戳
- 样本值（value）： 一个 folat64 的浮点型数据表示当前样本的值

### 表示方式
`<metric name>{<label name>=<label value>, ...}`

## 查询语句PromQL
　　Prometheus 提供了一种功能丰富的查询语言 PromQL，允许用户实时选择和汇聚时间序列数据。通过编写相应的表达式可以返回以下四种数据类型：
- 瞬时向量（Instant vector）
- 区间向量（Range vector）
- 标量（Scalar）
- 字符串（String）

　　具体 PromQL 用法可查询[Prometheus官方文档](https://prometheus.fuckcloudnative.io/di-san-zhang-prometheus/di-4-jie-cha-xun/basics)

## HTTP API
　　Prometheus 同样提供 HTTP API 调用接口，通过GET请求带入PromQL表达式、时间戳、时间间隔等参数，返回JSON格式的监控数据

### API响应格式

```
{
  "status": "success" | "error",
  "data": <data>,

  // Only set if status is "error". The data field may still hold
  // additional data.
  "errorType": "<string>",
  "error": "<string>"
}
```

### 表达式查询

　　通过 HTTP API 我们可以分别通过 /api/v1/query 和 /api/v1/query_range 查询 PromQL 表达式当前或者一定时间范围内的计算结果。

#### 瞬时数据查询

查询语句：
```
GET /api/v1/query?
	query=<string> &
	time=<rfc3339 | unix_timestamp>&timeout=<duration>
```

返回数据格式：
```
{
	"status" : "success",
	"data" : {
		"resultType": "matrix" | "vector" | "scalar" | "string",
		"result": <value>
	}
}
```

#### 区间数据查询

查询语句：
```
GET /api/v1/query_range?
	query=<string> &
	start=<rfc3339 | unix_timestamp> &
	end=<rfc3339 | unix_timestamp> &
	step=<duration | float> &
	timeout=<duration>
```

返回数据格式：
```
{
	"status" : "success",
  "data" : {
  	"resultType" : "matrix",
  	"result" : [
  	{
  		"metric": { "<label_name>": "<label_value>", ... },
  		"values": [ [ <unix_time>, "<sample_value>" ], ... ]
  	},
  	...
  	]
  }
}
```

---

# Grafana
　　Grafana是一款用Go语言开发的通用的开源数据可视化工具，可以做数据监控和数据统计，带有告警功能。“通用”意味着Grafana不仅仅适用于展示Prometheus下的监控数据，也同样适用于一些其他的数据可视化需求。
　　其主要特点如下：

- 可视化：快速灵活的客户端图表，面板插件有许多不同方式的可视化指标和日志，官方库中具有丰富的仪表盘插件，比如热图、折线图、图表等多种展示方式
- 报警：以可视方式定义最重要指标的警报规则，Grafana将不断计算并发送通知，在数据达到阈值时通过Slack、PagerDuty等获得通知
- 混合展示：在同一图表中混合使用不同的数据源，可以基于每个查询指定数据源，甚至自定义数据源
- 注释：使用来自不同数据源的丰富事件注释图表，将鼠标悬停在事件上会显示完整的事件元数据和标记
- 过滤器：Ad-hoc过滤器允许动态创建新的键/值过滤器，这些过滤器会自动应用于使用该数据源的所有查询

## 数据源（Data Source）
　　对于Grafana而言，Prometheus这类为其提供数据的对象均称为数据源（Data Source）。目前，Grafana官方提供了对： Prometheus, Graphite, InfluxDB, OpenTSDB等大量数据源的支持。对于Grafana管理员而言，只需要将这些对象以数据源的形式添加到Grafana中，Grafana便可以轻松的实现对这些数据的可视化工作。
　　与此同时，Grafana还支持使用第三方数据源插件导入数据，本次工作即通过使用 JSON 数据源插件引入后端自行处理过的数据进行展示。


<img src="/images/fig12.png" title="Grafana官方支持的部分数据源" height="650" style="display: inline-block">
<img src="/images/fig13.png" title="部分第三方插件数据源" height="650" style="display: inline-block">

## 仪表盘（Dashboard）

　　通过数据源定义好可视化的数据来源之后，对于用户而言最重要的事情就是实现数据的可视化。在Grafana中，我们通过Dashboard来组织和管理我们的数据可视化图表：
![](/images/fig14.png "Grafana Dashboard")

## 面板（Panel）
　　Panel是Grafana中最基本的可视化单元。每一种类型的面板都提供了相应的查询编辑器(Query Editor)，让用户可以从不同的数据源（如Prometheus）中查询出相应的监控数据，并且以可视化的方式展现。例如，如果以Prometheus作为数据源，那在Query Editor中，我们实际上使用的是PromQL，而Panel则会负责从特定的Prometheus中查询出相应的数据，并且将其可视化。由于每个Panel是完全独立的，因此在一个Dashboard中，往往可能会包含来自多个Data Source的数据。
　　在本次工作中，由于使用自行设计的后端接口，数据查询语句为自定义的JSON格式。

![](/images/fig15.png "官方支持的Panel类型（并支持插件扩展）")
![](/images/fig16.png "查询数据源与查询语句示例")


# JSON Datasource
　　一个用于Grafana的第三方JSON Datasource插件，通过编写后端实现该插件要求的HTTP API，可以实现以JSON格式向Grafana发送自定义的Datasource

![](/images/fig17.png "Json Data Sources")

## API
　　为使该插件正常工作，后端必须实现以下API：
- `GET /` ：用于测试与datasource之间的连接情况
- `POST /search` ：根据请求字段返回可用的metrics
- `POST /query` ：根据输入的查询语句返回对应的metrics
- `POST /annotations` ：返回注释（本次工作实际未实现，没有影响功能，故似乎为可选实现）

　　可选实现：
- `POST /tag-key` ：根据ad-hoc过滤器返回对应的tag-keys
- `POST /tag-values` ：根据ad-hoc过滤器返回对应的tag-values

## API 具体格式

### search

**request:**
Variables > New/Edit page 示例：
`{ "target": "query field value" }`
Panel > Queries page 示例：
`{ "type": "timeseries", "target": "upper_50" }`

**response:**
array返回示例：
`["upper_25","upper_50","upper_75","upper_90","upper_95"]`
map返回示例
`[ { "text": "upper_25", "value": 1}, { "text": "upper_75", "value": 2} ]`

### query

**request:**
timeseries 示例：
```
{
  "panelId": 1,
  "range": {
    "from": "2016-10-31T06:33:44.866Z",
    "to": "2016-10-31T12:33:44.866Z",
    "raw": {
      "from": "now-6h",
      "to": "now"
    }
  },
  "rangeRaw": {
    "from": "now-6h",
    "to": "now"
  },
  "interval": "30s",
  "intervalMs": 30000,
  "maxDataPoints": 550,
  "targets": [
     { "target": "Packets", "refId": "A", "type": "timeseries", "data": { "additional": "optional json" } },
     { "target": "Errors", "refId": "B", "type": "timeseries" }
  ],
  "adhocFilters": [{
    "key": "City",
    "operator": "=",
    "value": "Berlin"
  }]
}
```


Additional data：
request可携带额外的JSON格式的数据作为查询条件，格式如下：
`{ "additional": "optional json" }`
并将放置在request body 的`target-data`中，示例如下：
`{ "target": "upper_50", "refId": "A", "type": "timeseries", "data": { "additional": "optional json" } }`

**response:**
metric类型选择为timeseries 返回示例： (metric类型为float，unix时间戳类型为milliseconds)
```
[
  {
    "target":"pps in",
    "datapoints":[
      [622,1450754160000],
      [365,1450754220000]
    ]
  },
  {
    "target":"pps out",
    "datapoints":[
      [861,1450754160000],
      [767,1450754220000]
    ]
  },
  {
    "target":"errors out",
    "datapoints":[
      [861,1450754160000],
      [767,1450754220000]
    ]
  },
  {
    "target":"errors in",
    "datapoints":[
      [861,1450754160000],
      [767,1450754220000]
    ]
  }
]
```

metric类型选择为table 返回示例：
```
[
  {
    "columns":[
      {"text":"Time","type":"time"},
      {"text":"Country","type":"string"},
      {"text":"Number","type":"number"}
    ],
    "rows":[
      [1234567,"SE",123],
      [1234567,"DE",231],
      [1234567,"US",321]
    ],
    "type":"table"
  }
]
```

*其余API接口本次未使用，便不再记录*

# 参考学习资料
>[Prometheus中文文档](https://prometheus.fuckcloudnative.io/)
>[实战 Prometheus 搭建监控系统](https://www.aneasystone.com/archives/2018/11/prometheus-in-action.html)
>[可视化工具Grafana：简介及安装](https://www.cnblogs.com/imyalost/p/9873641.html)
>[JSON Datasource – a generic backend datasource](https://github.com/simPod/grafana-json-datasource)


*GO后端实现方案与GO学习路径再一同总结*