---
layout:     post
rewards: false
title:      opentelemetry
categories:
    - k8s


---

OpenTelemetry 的目的是收集、处理和输出信号。 信号是用于描述操作系统和平台上运行的应用程序基本活动的系统输出。信号可以是你想在特定时间点测量的东西，如温度或内存使用率，也可以是你想追踪的分布式系统组件中发生的事件。你可以将不同的信号组合在一起，从不同角度观察同一项技术的内部运作。



OpenTelemetry 提供了一种捕获可观测性信号的标准化方法：

- Metrics(指标) 表明存在问题。
- Traces(跟踪) 会告诉你问题出在哪里。
- Logs(日志) 可以帮助您找到根本原因。



OpenTelemetry 目前支持 [traces](https://opentelemetry.io/docs/concepts/signals/traces/)、[metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)、[logs](https://opentelemetry.io/docs/concepts/signals/logs/)和[baggage](https://opentelemetry.io/docs/concepts/signals/baggage/)。

- **trace**：在分布式应用程序中的完整请求链路信息。

  这是根**span** (跨度)，表示整个操作的开始和结束。请注意，它有一个`trace_id`表示跟踪的字段，但没有 `parent_id`。这就是您知道它是根跨度的原因。

  ```json
  {
    "name": "hello",
    "context": {
      "trace_id": "5b8aa5a2d2c872e8321cf37308d69df2",
      "span_id": "051581bf3cb55c13"
    },
    "parent_id": null,  // 填上一个span_id
    "start_time": "2022-04-29T18:52:58.114201Z",
    "end_time": "2022-04-29T18:52:58.114687Z",
    "attributes": {
      "http.route": "some_route1"
    },
    "events": [
      {
        "name": "Guten Tag!",
        "timestamp": "2022-04-29T18:52:58.114561Z",
        "attributes": {
          "event_attributes": 1
        }
      }
    ]
  }
  ```

- **指标**是在运行时捕获的服务指标，应用程序和请求指标是可用性和性能的重要指标。

  指标的定义如下：

  - 姓名、种类、单位（可选）、描述（可选）
  - 指标种类和Prometheus差不多。

  

- **日志**是系统或应用程序在特定时间点发生的事件的文本记录。

- **baggage**：是在信号之间传递的上下文信息。

  Baggage 是位于 context 旁边的上下文信息。Baggage 是一个键值存储，这意味着它允许您 与[context一起](https://opentelemetry.io/docs/concepts/context-propagation/#context)[传播](https://opentelemetry.io/docs/concepts/context-propagation/#propagation)任何您喜欢的数据。





Run the application like before, but don’t export to the console:

```shell
export OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true
opentelemetry-instrument --logs_exporter otlp flask run -p 8080
```

By default, `opentelemetry-instrument` exports traces and metrics over OTLP/gRPC and will send them to `localhost:4317`, which is what the collector is listening on.

When you access the `/rolldice` route now, you’ll see output in the collector process instead of the flask process, which should look something like this: