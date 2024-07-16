修改nginx-ingress-controller DaemonSet  参考https://10.19.64.205:3443/dashboard/c/c-8qszh/explorer/apps.daemonset/ingress-nginx/nginx-ingress-controller?mode=edit&as=yaml

```yaml
  annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"

	ports:
  - containerPort: 10254
  	name: prometheus
  	protocol: TCP
  	
    volumeMounts:
    - mountPath: /var/log/nginx
    name: shared-data

# sidecar 共享access.log
- image: nginx/prometheus-nginxlog-exporter:v1.10.0
	args:
	- /mnt/nginxlogs/access.log
  imagePullPolicy: IfNotPresent
  name: exporter
  ports:
  - containerPort: 4040
  	name: exporter
  	protocol: TCP
  volumeMounts:
  - mountPath: /mnt/nginxlogs
  	name: shared-data
  
	volumes:
  - emptyDir: {}
  	name: shared-data
```

配置accesslog format  参考https://10.19.64.205:3443/dashboard/c/c-8qszh/explorer/configmap/ingress-nginx/ingress-nginx-controller?as=yaml#data

配置**configmap**:ingress-nginx-controller  **ns**:ingress-nginx，ingres**s默认配置文件**不能随便改

```yaml
apiVersion: v1
data:
  log-format-upstream: $remote_addr - $remote_user [$time_local] "$request" $status
    $body_bytes_sent "$http_referer" "$http_user_agent" "$http_x_forwarded_for"
kind: ConfigMap
```



创建Service  为了给  **4040,10254端口**创建服务，label `k8s-app: ingress-nginx`

参考https://10.19.64.205:3443/dashboard/c/c-8qszh/explorer/service/ingress-nginx/ingress-nginx#pods

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    field.cattle.io/creatorId: user-hdngh
    field.cattle.io/ipAddresses: "null"
    field.cattle.io/publicEndpoints: '[{"addresses":["10.81.17.131"],"port":30284,"protocol":"TCP","serviceName":"ingress-nginx:ingress-nginx","allNodes":true},{"addresses":["10.81.17.131"],"port":30771,"protocol":"TCP","serviceName":"ingress-nginx:ingress-nginx","allNodes":true}]'
    field.cattle.io/targetDnsRecordIds: "null"
    field.cattle.io/targetWorkloadIds: '["daemonset:ingress-nginx:nginx-ingress-controller"]'
    prometheus.io/port: '"10254"'
    prometheus.io/scrape: '"true"'
  creationTimestamp: "2022-12-13T06:36:40Z"
  labels:
    app: ingress-nginx
    cattle.io/creator: norman
    k8s-app: ingress-nginx
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .: {}
          f:field.cattle.io/creatorId: {}
          f:field.cattle.io/ipAddresses: {}
          f:field.cattle.io/targetDnsRecordIds: {}
          f:field.cattle.io/targetWorkloadIds: {}
          f:prometheus.io/port: {}
          f:prometheus.io/scrape: {}
        f:labels:
          .: {}
          f:app: {}
          f:cattle.io/creator: {}
          f:k8s-app: {}
      f:spec:
        f:externalTrafficPolicy: {}
        f:internalTrafficPolicy: {}
        f:ports:
          .: {}
          k:{"port":10254,"protocol":"TCP"}:
            .: {}
            f:name: {}
            f:port: {}
            f:protocol: {}
            f:targetPort: {}
        f:sessionAffinity: {}
        f:type: {}
    manager: rancher
    operation: Update
    time: "2022-12-20T08:36:54Z"
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          f:field.cattle.io/publicEndpoints: {}
      f:spec:
        f:ports:
          k:{"port":4040,"protocol":"TCP"}:
            .: {}
            f:name: {}
            f:nodePort: {}
            f:port: {}
            f:protocol: {}
            f:targetPort: {}
        f:selector: {}
    manager: agent
    operation: Update
    time: "2022-12-22T08:02:17Z"
  name: ingress-nginx
  namespace: ingress-nginx
  resourceVersion: "53475608"
  uid: 646ddaf3-f768-43e0-8bbc-041b76b65f4b
spec:
  clusterIP: 10.43.206.129
  clusterIPs:
  - 10.43.206.129
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: prometheus
    nodePort: 30284
    port: 10254
    protocol: TCP
    targetPort: 10254
  - name: stubstatus
    nodePort: 30771
    port: 4040
    protocol: TCP
    targetPort: 4040
  selector:
    workloadID_ingress-nginx: "true"
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}

```



创建ServiceMonitor

https://10.19.64.205:3443/dashboard/c/c-8qszh/monitoring/monitor/ingress-nginx/ingress-nginx?resource=monitoring.coreos.com.servicemonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    k8s-apps: ingress-nginx
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  endpoints:
  - interval: 10s
    port: prometheus
  - interval: 10s
    port: stubstatus
  jobLabel: k8s-app
  namespaceSelector:
    matchNames:
    - ingress-nginx
  selector:
    matchLabels:
      k8s-app: ingress-nginx
```



ingress 指标

```sh
Ingress当前客户端连接数 nginx_ingress_controller_nginx_process_connections


算速率例子
rate(nginx_ingress_controller_nginx_process_requests_total[10m])

流量速率
nginx_ingress_controller_nginx_process_read_bytes_total
nginx_ingress_controller_nginx_process_write_bytes_total

state 区分
客户端连接速率 nginx_ingress_controller_nginx_process_connections_total

QPS  nginx_ingress_controller_nginx_process_requests_total

成功率2xx,3xx
sum(nginx_http_response_count_total{status=~"3..|2.."})/sum(nginx_http_response_count_total)
失败
sum(nginx_http_response_count_total{status!~"3..|2.."})/sum(nginx_http_response_count_total)


```

