---
layout:     post
rewards: false
title:      Prometheus Exporter 详解
categories:
    - k8s
---

- [自定义 demo](http://www.xuyasong.com/?p=1942)
- [redis exporter](https://github.com/oliver006/redis_exporter)

- [官方exporter方法介绍](https://prometheus.io/docs/instrumenting/writing_clientlibs/)

# 自定义Exporter

主要是要实现Collector的接口`Describe` 和`Collect`

```go
type Collector interface {
	// Describe sends the super-set of all possible descriptors of metrics
	// collected by this Collector to the provided channel and returns once
	// the last descriptor has been sent. The sent descriptors fulfill the
	// consistency and uniqueness requirements described in the Desc
	// documentation.
	//
	// It is valid if one and the same Collector sends duplicate
	// descriptors. Those duplicates are simply ignored. However, two
	// different Collectors must not send duplicate descriptors.
	//
	// Sending no descriptor at all marks the Collector as “unchecked”,
	// i.e. no checks will be performed at registration time, and the
	// Collector may yield any Metric it sees fit in its Collect method.
	//
	// This method idempotently sends the same descriptors throughout the
	// lifetime of the Collector. It may be called concurrently and
	// therefore must be implemented in a concurrency safe way.
	//
	// If a Collector encounters an error while executing this method, it
	// must send an invalid descriptor (created with NewInvalidDesc) to
	// signal the error to the registry.
	Describe(chan<- *Desc)
	// Collect is called by the Prometheus registry when collecting
	// metrics. The implementation sends each collected metric via the
	// provided channel and returns once the last metric has been sent. The
	// descriptor of each sent metric is one of those returned by Describe
	// (unless the Collector is unchecked, see above). Returned metrics that
	// share the same descriptor must differ in their variable label
	// values.
	//
	// This method may be called concurrently and must therefore be
	// implemented in a concurrency safe way. Blocking occurs at the expense
	// of total performance of rendering all registered metrics. Ideally,
	// Collector implementations support concurrent readers.
	Collect(chan<- Metric)
}
```

Describe Exceple  告诉 `prometheus` 我们定义了哪些 `prometheus.Desc` 结构，通过 `channel` 传递给上层

```go
func newMetricDesc(namespace string, metricName string, help string, labels []string) *prometheus.Desc {
	return prometheus.NewDesc(prometheus.BuildFQName(namespace, "", metricName), help, labels, nil)
}

totalScrapes =: prometheus.NewCounter(prometheus.CounterOpts{
  Namespace: opts.Namespace,
  Name:      "exporter_scrapes_total",
  Help:      "Current total redis scrapes.",
})

scrapeDuration:= prometheus.NewSummary(prometheus.SummaryOpts{
  Namespace: opts.Namespace,
  Name:      "exporter_scrape_duration_seconds",
  Help:      "Durations of scrapes by the exporter",
})

func (e *Exporter) Describe(ch chan<- *prometheus.Desc) {
  
  // 把Desc放到ch
	for _, desc := range e.metricDescriptions {
		ch <- desc
	}

	for _, v := range e.metricMapGauges {
		ch <- newMetricDesc(e.options.Namespace, v, v+" metric", nil)
	}

	for _, v := range e.metricMapCounters {
		ch <- newMetricDesc(e.options.Namespace, v, v+" metric", nil)
	}

	ch <- e.totalScrapes.Desc()
	ch <- e.scrapeDuration.Desc()

}
```

Collect Example  真正实现数据采集的功能，将采集数据结果通过 channel 传递给上层

```go
func (e *Exporter) registerConstMetric(ch chan<- prometheus.Metric, metric string, val float64, valType prometheus.ValueType, labelValues ...string) {
	descr := e.metricDescriptions[metric]
	if descr == nil {
		descr = newMetricDescr(e.options.Namespace, metric, metric+" metric", labelValues)
	}
  // 主要是为valType=GaugeValue,CounterValue 之类赋值，不同类型赋值不一样
	if m, err := prometheus.NewConstMetric(descr, valType, val, labelValues...); err == nil {
		ch <- m
	}
}

func (e *Exporter) Collect(ch chan<- prometheus.Metric) {
	e.Lock()
	defer e.Unlock()
  // 原子操作
  /*
  func (c *counter) Inc() {
		atomic.AddUint64(&c.valInt, 1)
	}
  */
	e.totalScrapes.Inc()

	if e.redisAddr != "" {
		startTime := time.Now()
    ......

		e.registerConstMetricGauge(ch, "up", up)

		took := time.Since(startTime).Seconds()
		e.scrapeDuration.Observe(took)
		e.registerConstMetricGauge(ch, "exporter_last_scrape_duration_seconds", took)
	}

	ch <- e.totalScrapes
	ch <- e.scrapeDuration
	ch <- e.targetScrapeRequestErrors
}
```

main example

```go
var (
    // 命令行参数
    listenAddr       = flag.String("web.listen-port", "9002", "An port to listen on for web interface and telemetry.")
    metricsPath      = flag.String("web.telemetry-path", "/metrics", "A path under which to expose metrics.")
    metricsNamespace = flag.String("metric.namespace", "bec", "Prometheus metrics namespace, as the prefix of metrics name")
)

func main() {
    flag.Parse()
    // 初始化自定义Collter
    demoExporter, _ := exporter.NewDemoExporter()
    prometheus.MustRegister(demoExporter)
    http.Handle(*metricPath, promhttp.Handler())

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte(`<html>
            <head><title>A Prometheus Exporter</title></head>
            <body>
            <h1>A Prometheus Exporter</h1>
            <p><a href='/metrics'>Metrics</a></p>
            </body>
            </html>`))
    })

    log.Printf("Starting Server at http://localhost:%s%s", *listenAddr, *metricsPath)
    log.Fatal(http.ListenAndServe(":" + *listenAddr, nil))
}
```

# vec/summary/history 用法

- [官方example](https://pkg.go.dev/github.com/prometheus/client_golang/prometheus?utm_source=godoc#pkg-examples)

- [metric_types](https://prometheus.io/docs/concepts/metric_types/)只看这个根本看不懂说的啥



CounterVec是一组counter，这些计数器具有相同的描述，但它们的变量标签具有不同的值。 如果要计算按各种维度划分的相同内容

```go
//step1:初始化一个容器
httpReqs := prometheus.NewCounterVec(
    prometheus.CounterOpts{
        Name: "http_requests_total",
        Help: "How many HTTP requests processed, partitioned by status code and HTTP method.",
    },
    []string{"code", "method"},
)
//step2:注册容器
prometheus.MustRegister(httpReqs)

httpReqs.WithLabelValues("404", "POST").Add(42)

// If you have to access the same set of labels very frequently, it
// might be good to retrieve the metric only once and keep a handle to
// it. But beware of deletion of that metric, see below!
//step3:向容器中写入值，主要调用容器的方法如Inc()或者Add()方法
m := httpReqs.WithLabelValues("200", "GET")
for i := 0; i < 1000000; i++ {
    m.Inc()
}
// Delete a metric from the vector. If you have previously kept a handle
// to that metric (as above), future updates via that handle will go
// unseen (even if you re-create a metric with the same label set
// later).
httpReqs.DeleteLabelValues("200", "GET")
// Same thing with the more verbose Labels syntax.
httpReqs.Delete(prometheus.Labels{"method": "GET", "code": "200"})
```

要一次性统计四个cpu的温度，这个时候就适合使用GaugeVec了

```go
cpusTemprature := prometheus.NewGaugeVec(
    prometheus.GaugeOpts{
        Name:      "CPUs_Temperature",
        Help:      "the temperature of CPUs.",
    },
    []string{
        // Which cpu temperature?
        "cpuName",
    },
)
prometheus.MustRegister(cpusTemprature)

cpusTemprature.WithLabelValues("cpu1").Set(temperature1)
cpusTemprature.WithLabelValues("cpu2").Set(temperature2)
cpusTemprature.WithLabelValues("cpu3").Set(temperature3)
```

summary

```go
temps := prometheus.NewSummary(prometheus.SummaryOpts{
  Name:       "pond_temperature_celsius",
  Help:       "The temperature of the frog pond.",
  Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
})

// Simulate some observations.
for i := 0; i < 1000; i++ {
  temps.Observe(30 + math.Floor(120*math.Sin(float64(i)*0.1))/10)
}

// Just for demonstration, let's check the state of the summary by
// (ab)using its Write method (which is usually only used by Prometheus
// internally).
metric := &dto.Metric{}
temps.Write(metric)
fmt.Println(proto.MarshalTextString(metric))

summary: <
  sample_count: 1000
  sample_sum: 29969.50000000001
  quantile: <
    quantile: 0.5
    value: 31.1
  >
  quantile: <
    quantile: 0.9
    value: 41.3
  >
  quantile: <
    quantile: 0.99
    value: 41.9
  >
>
```

Histogram

```go
temps := prometheus.NewHistogram(prometheus.HistogramOpts{
    Name:    "pond_temperature_celsius",
    Help:    "The temperature of the frog pond.", // Sorry, we can't measure how badly it smells.
    Buckets: prometheus.LinearBuckets(20, 5, 5),  // 5 buckets, each 5 centigrade wide.
})

// Simulate some observations.
for i := 0; i < 1000; i++ {
    temps.Observe(30 + math.Floor(120*math.Sin(float64(i)*0.1))/10)
}

// Just for demonstration, let's check the state of the histogram by
// (ab)using its Write method (which is usually only used by Prometheus
// internally).
metric := &dto.Metric{}
temps.Write(metric)
fmt.Println(proto.MarshalTextString(metric))


histogram: <
  sample_count: 1000
  sample_sum: 29969.50000000001
  bucket: <
    cumulative_count: 192
    upper_bound: 20
  >
  bucket: <
    cumulative_count: 366
    upper_bound: 25
  >
  bucket: <
    cumulative_count: 501
    upper_bound: 30
  >
  bucket: <
    cumulative_count: 638
    upper_bound: 35
  >
  bucket: <
    cumulative_count: 816
    upper_bound: 40
  >
>

```







# 采集源码分析

主要是NewRegistry返回Registry，分别实现了**Registerer**和**Gatherer**的接口

![image-20211120120812861](https://tva1.sinaimg.cn/large/008i3skNgy1gwlhkq42qfj31360g0jtj.jpg)

**调用注册时候调用Collector.Describe**

```go
// Register implements Registerer.
func (r *Registry) Register(c Collector) error {
	var (
		descChan           = make(chan *Desc, capDescChan)
		newDescIDs         = map[uint64]struct{}{}
		newDimHashesByName = map[string]uint64{}
		collectorID        uint64 // All desc IDs XOR'd together.
		duplicateDescErr   error
	)
	go func() {
		c.Describe(descChan)
		close(descChan)
	}()
	r.mtx.Lock()
	defer func() {
		// Drain channel in case of premature return to not leak a goroutine.
		for range descChan {
		}
		r.mtx.Unlock()
	}()
	// Conduct various tests...
	for desc := range descChan {
    .....//保存Desc信息
  }
}
```

Prometheus Server pull 时候会调用`/MetricsPath`的handler

```go
// HandlerFor returns an uninstrumented http.Handler for the provided
// Gatherer. The behavior of the Handler is defined by the provided
// HandlerOpts. Thus, HandlerFor is useful to create http.Handlers for custom
// Gatherers, with non-default HandlerOpts, and/or with custom (or no)
// instrumentation. Use the InstrumentMetricHandler function to apply the same
// kind of instrumentation as it is used by the Handler function.
func HandlerFor(reg prometheus.Gatherer, opts HandlerOpts) http.Handler {
	....
  mfs, err := reg.Gather()
	....
  // 返回OpenMetrics
}
```

**里面会调用Registry.Gather实现**，pull的时候才执行**collector.Collect**

```go
// Gather implements Gatherer.
func (r *Registry) Gather() ([]*dto.MetricFamily, error) {
  ....
  r.mtx.RLock()
  ....
  for _, collector := range r.collectorsByID {
		checkedCollectors <- collector
	}
  r.mtx.RUnlock()
  ...
  wg.Add(goroutineBudget)

	collectWorker := func() {
		for {
			select {
			case collector := <-checkedCollectors:
				collector.Collect(checkedMetricChan)
			case collector := <-uncheckedCollectors:
				collector.Collect(uncheckedMetricChan)
			default:
				return
			}
			wg.Done()
		}
	}

	// Start the first worker now to make sure at least one is running.
	go collectWorker()
	goroutineBudget--
  
  // Close checkedMetricChan and uncheckedMetricChan once all collectors
	// are collected.
	go func() {
		wg.Wait()
		close(checkedMetricChan)
		close(uncheckedMetricChan)
	}()
  ......
  cmc := checkedMetricChan
	umc := uncheckedMetricChan

	for {
		select {
		case metric, ok := <-cmc:
			if !ok {
				cmc = nil
				break
			}
			errs.Append(processMetric(
				metric, metricFamiliesByName,
				metricHashes,
				registeredDescIDs,
			))
  
}
```

然后通过**processMetric**里面`metric.Write`输出指标值之类

```go
// processMetric is an internal helper method only used by the Gather method.
func processMetric(
	metric Metric,
	metricFamiliesByName map[string]*dto.MetricFamily,
	metricHashes map[uint64]struct{},
	registeredDescIDs map[uint64]struct{},
) error {
	desc := metric.Desc()
	// Wrapped metrics collected by an unchecked Collector can have an
	// invalid Desc.
	if desc.err != nil {
		return desc.err
	}
	dtoMetric := &dto.Metric{}
	if err := metric.Write(dtoMetric); err != nil {
		return fmt.Errorf("error collecting metric %v: %s", desc, err)
	}
	.....
]
```

# 编译成docker image

```dockerfile
FROM xxx/go/ubuntu_builder AS builder
ARG WORK_DIR

# go build
WORKDIR ${WORK_DIR}
COPY go.* ${WORK_DIR}/
RUN go version && go mod download
COPY . ${WORK_DIR}
# 使用静态编译不然使用scratch会报错  standard_init_linux.go:228: exec user process caused: no such file or directory
RUN CGO_ENABLED=0 go build -ldflags "-w -s -extldflags \"-static\" " -o client
#RUN go build -ldflags "-w -s " -o client

FROM scratch
ARG WORK_DIR

COPY --from=builder ${WORK_DIR}/client /exporter

EXPOSE 9121
ENTRYPOINT [ "/exporter" ]
```



```bash
#!/bin/bash

set -ex

HUB_REPO="xxxx"
IMAGE_NAME=${HUB_REPO}/exporter/demo_exporter
WORK_DIR=/exporter

docker build --build-arg WORK_DIR=${WORK_DIR} -t ${IMAGE_NAME} -f Dockerfile .
#docker run -it ${IMAGE_NAME} /bin/sh
docker push ${IMAGE_NAME}
```

## golang静态编译

golang 的编译（不涉及 cgo 编译的前提下）默认使用了静态编译，不依赖任何动态链接库。

这样可以任意部署到各种运行环境，不用担心依赖库的版本问题。只是体积大一点而已，存储时占用了一点磁盘，运行时，多占用了一点内存。

- [Go 语言镜像精简](http://dockerone.com/article/10354)

静态编译与动态编译的区别
　　动态编译的 可执行文件 需要附带一个的 动态链接库 ，在执行时，需要调用其对应动态链接库中的命令。所以其优点一方面是缩小了执行文件本身的体积，另一方面是加快了编译速度，节省了 系统资源 。缺点一是哪怕是很简单的程序，只用到了链接库中的一两条命令，也需要附带一个相对庞大的链接库；二是如果其他计算机上没有安装对应的 运行库 ，则用动态编译的可执行文件就不能运行。
　　静态编译就是编译器在编译可执行文件的时候，将可执行文件需要调用的对应动态链接库(.so)中的部分提取出来，链接到可执行文件中去，使可执行文件在运行的时候不依赖于动态链接库。所以其优缺点与动态编译的可执行文件正好互补。

## docker镜像精简

- scratch：空镜像，基础镜像

  scratch是Docker中预留的最小的基础镜像。bosybox 、 Go语言编译打包的镜像都可以基于scratch来构建。

-  busybox

   busybox镜像只有几兆。

   BusyBox是一个集成了一百多个最常用Linux命令和工具（如cat、echo、grep、mount、telnet等）的精简工具箱，它只有几MB的大小，很方便进行各种快速验证，被誉为“Linux系统的瑞士军刀”。BusyBox可运行于多款POSIX环境的操作系统中，如Linux（包括Android）、Hurd、FreeBSD等。

- Alpine

  Alpine镜像比busybox大一点，也只有几兆。

  Alpine操作系统是一个面向安全的轻型Linux发行版。它不同于通常的Linux发行版，Alpine采用了musl libc和BusyBox以减小系统的体积和运行时资源消耗，但功能上比BusyBox又完善得多。在保持瘦身的同时，Alpine还提供了自己的包管理工具apk，可以通过https://pkgs.alpinelinux.org/packages查询包信息，也可以通过apk命令直接查询和安装各种软件。

  Alpine Docker镜像也继承了Alpine Linux发行版的这些优势。相比于其他Docker镜像，它的容量非常小，仅仅只有5MB左右（Ubuntu系列镜像接近200MB），且拥有非常友好的包管理机制。官方镜像来自docker-alpine项目。

  目前Docker官方已开始推荐使用Alpine替代之前的Ubuntu作为基础镜像环境。这样会带来多个好处，包括镜像下载速度加快，镜像安全性提高，主机之间的切换更方便，占用更少磁盘空间等。



# 部署

可以看到exporter其实就是个http server，**可以使用k8s的Deployment部署，通过arg或者env传参启动exporter**

```go
// /health  
func (e *Exporter) healthHandler(w http.ResponseWriter, r *http.Request) {
	_, _ = w.Write([]byte(`ok`))
}

// 首页 /
func (e *Exporter) indexHandler(w http.ResponseWriter, r *http.Request) {
	_, _ = w.Write([]byte(`<html>
<head><title>Redis Exporter ` + e.buildInfo.Version + `</title></head>
<body>
<h1>Redis Exporter ` + e.buildInfo.Version + `</h1>
<p><a href='` + e.options.MetricsPath + `'>Metrics</a></p>
</body>
</html>
`))
}
```

一个Service可以公开一个或多个服务端口,通常情况下,这些端口由指向一个Pod的多个Endpoints支持。这也反映在各自的Endpoints对象中。

Prometheus Operator引入ServiceMonitor对象, 它发现Endpoints对象并配置Prometheus去监控这些Pods。通过标签匹配到对应的Service，调用对应对应的服务

ServiceMonitorSpec的endpoints部分用于配置需要收集metrics的Endpoints的端口和其他参数。在一些用例中会直接监控不
在服务endpoints中的pods的端口。因此,在endpoints部分指定endpoint时,请严格使用,不要混淆。



ServiceMonitor和发现的目标可能来自任何namespace。这对于跨namespace的监控十分重要,比如meta-monitoring。使用
PrometheusSpec下ServiceMonitor Namespace Selectorn,通过各自Prometheus server限制ServiceMonitors作用namespece。使用ServiceMonitorSpec下的namespaceSelector可以现在允许发现Endpoints对象的命名空间。要发现所有命
名空间下的目标,namespaceSelector必须为空。


```yaml
{{- if .Values.prometheus.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "xxxx.fullname" . }}
  {{- if .Values.prometheus.serviceMonitor.namespace }}
  namespace: {{ .Values.prometheus.serviceMonitor.namespace }}
  {{- end }}
  labels:
    app.kubernetes.io/name: {{ include "xxx.name" . }} # 带有同样label service
    {{- if .Values.labels -}}
    {{ .Values.labels | toYaml | nindent 4 -}}
    {{- end }}
spec:
  jobLabel: jobLabel
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "xxx.name" . }} # 带有同样label service
  namespaceSelector:
    matchNames:
      - {{ .Release.Namespace }}
  endpoints:
    - port: metrics  # pull端口
      interval: {{ .Values.prometheus.serviceMonitor.interval }} # pull时间间隔
      {{- if .Values.prometheus.serviceMonitor.scrapeTimeout }}
      scrapeTimeout: {{ .Values.prometheus.serviceMonitor.scrapeTimeout }}
      {{- end }}
      {{- if .Values.prometheus.serviceMonitor.metricRelabelings }}
      metricRelabelings:
      {{- toYaml .Values.prometheus.serviceMonitor.metricRelabelings | nindent 4 }}
  {{- end }}
  {{- end }}
  
----
apiVersion: v1
kind: Service
metadata:
  name: {{ include "xxx.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "xxx.name" . }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: metrics
      protocol: TCP
      name: metrics
  selector:
    app.kubernetes.io/name: {{ include "xxx.name" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}


----
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "kafka-exporter.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "kafka-exporter.name" . }}
    helm.sh/chart: {{ include "kafka-exporter.chart" . }}
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: {{ .Release.Service }}
    {{- if .Values.labels -}}
    {{ .Values.labels | toYaml | nindent 4 -}}
    {{- end }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ include "kafka-exporter.name" . }}
      app.kubernetes.io/instance: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ include "kafka-exporter.name" . }}
        app.kubernetes.io/instance: {{ .Release.Name }}
        {{- if .Values.podLabels -}}
        {{ .Values.podLabels | toYaml | nindent 8 -}}
        {{- end }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          args:
            {{- if .Values.kafkaExporter}}
            {{- range .Values.kafkaExporter.kafka.servers }}
            - "--kafka.server={{ . }}"
            {{- end }}
            {{- if .Values.kafkaExporter.kafka.version }}
            - --kafka.version={{ .Values.kafkaExporter.kafka.version }}
            {{- end }}
            {{- end}}
            {{- if .Values.kafkaExporter.sasl.enabled }}
            - --sasl.enabled
            {{- if not .Values.kafkaExporter.sasl.handshake }}
            - --sasl.handshake=false
            {{- end }}
            - --sasl.username={{ .Values.kafkaExporter.sasl.username }}
            - --sasl.password={{ .Values.kafkaExporter.sasl.password }}
            - --sasl.mechanism={{ .Values.kafkaExporter.sasl.mechanism }}
            {{- end }}
            {{- if .Values.kafkaExporter.tls.enabled}}
            - --tls.enabled
            {{- if .Values.kafkaExporter.tls.insecureSkipTlsVerify}}
            - --tls.insecure-skip-tls-verify
            {{- else }}
            - --tls.ca-file=/etc/tls-certs/ca-file
            - --tls.cert-file=/etc/tls-certs/cert-file
            - --tls.key-file=/etc/tls-certs/key-file
            {{- end }}
            {{- end }}
            {{- if .Values.kafkaExporter.log }}
            - --verbosity={{ .Values.kafkaExporter.log.verbosity }}
            {{- end }}
            {{- if .Values.kafkaExporter.log.enableSarama }}
            - --log.enable-sarama
            {{- end }}
          ports:
            - name: metrics
              containerPort: 9308
              protocol: TCP
          livenessProbe:
            failureThreshold: 1
            httpGet:
              path: /healthz
              port: metrics
              scheme: HTTP
            initialDelaySeconds: 3
            periodSeconds: 30
            successThreshold: 1
            timeoutSeconds: 9
          readinessProbe:
            failureThreshold: 1
            httpGet:
              path: /healthz
              port: metrics
              scheme: HTTP
            initialDelaySeconds: 3
            periodSeconds: 15
            successThreshold: 1
            timeoutSeconds: 9

          {{- if and .Values.kafkaExporter.tls.enabled (not .Values.kafkaExporter.tls.insecureSkipTlsVerify) }}
          volumeMounts:
          - name: tls-certs
            mountPath: "/etc/tls-certs/"
            readOnly: true
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}

      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
    {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
    {{- end }}
    {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
    {{- end }}
    {{- if and .Values.kafkaExporter.tls.enabled (not .Values.kafkaExporter.tls.insecureSkipTlsVerify) }}
      volumes:
      - name: tls-certs
        secret:
          secretName: {{ include "kafka-exporter.fullname" . }}
    {{- end }}



----
# values
prometheus:
  serviceMonitor:
    enabled: true
    namespace: monitoring
    interval: "30s"
    additionalLabels:
      app: kafka-exporter
    metricRelabelings: {}
```

可以看到新发现的target

![img](https://tva1.sinaimg.cn/large/008i3skNgy1gwlmrsatajj314q0ajdhj.jpg)



# 添加到Grafana

https://blog.csdn.net/hjxzb/article/details/81044583



# 实例代码

https://github.com/631068264/demo_exporter

