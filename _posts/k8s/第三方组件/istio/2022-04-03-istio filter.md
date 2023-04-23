---
layout:     post
rewards: false
title:   Istio filter
categories:
    - k8s
---

参考

- https://istio.io/latest/docs/reference/config/networking/envoy-filter/
- https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/lua_filter#lua
- https://github.com/envoyproxy/envoy
- https://github.com/envoyproxy/data-plane-api

# 术语

**主机**：能够进行网络通信的实体（手机、服务器等应用程序）。在本文档中，主机是一个逻辑网络应用程序。一个物理硬件可能有多个主机在其上运行，只要它们中的每一个都可以独立寻址。

**downstream**：下游主机连接到 Envoy，发送请求并接收响应。

**upstream**：上游主机接收来自 Envoy 的连接和请求并返回响应。

**侦听器**：侦听器是一个命名的网络位置（例如，端口、unix 域套接字等），下游客户端可以连接到该位置。Envoy 公开一个或多个下游主机连接的侦听器。

**集群**：集群是 Envoy 连接到的一组逻辑相似的上游主机。[Envoy 通过服务发现发现](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/service_discovery#arch-overview-service-discovery)集群的成员。它可以选择通过[主动健康检查](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/health_checking#arch-overview-health-checking)来确定集群成员的健康状况。Envoy 将请求路由到的集群成员由[负载平衡策略](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/overview#arch-overview-load-balancing)确定。

**Mesh**：一组协调以提供一致的网络拓扑的主机。在本文档中，“Envoy 网格”是一组 Envoy 代理，它们为由许多不同服务和应用程序平台组成的分布式系统形成消息传递基础。

**运行时配置**：与 Envoy 一起部署的带外实时配置系统。可以更改影响操作的配置设置，而无需重新启动 Envoy 或更改主要配置。





# lua 配置

By default, Lua script defined in `inline_code` will be treated as a `GLOBAL` script. Envoy will execute it for every HTTP request.

- httpCall调用其他namespace的service
- [Accessing request headers in envoy_on_response (lua HTTP filter)](https://github.com/envoyproxy/envoy/issues/4613)使用lua filter在response获取request请求头
- 使用[httpbin调试](http://httpbin.org/#/Anything/post_anything__anything_)



```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: ums-filter
  namespace: istio-system
spec:
  configPatches:
    - applyTo: HTTP_FILTER # 将补丁应用于 http 连接管理器中的 HTTP 过滤器链，以修改现有过滤器或添加新过滤器
      listener:
        filterChain:
          filter:
            name: envoy.http_connection_manager #具有多个过滤器链的侦听器（例如，具有允许 mTLS 的边车上的入站侦听器，具有多个 SNI 匹配的网关侦听器），过滤器链匹配可用于选择特定的过滤器链进行修补
            subFilter:
              name: ""
      match:
        context: GATEWAY
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.lua
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua
            inline_code: |
              -- By default, Lua script defined in inline_code will be treated as a GLOBAL script. Envoy will execute it for every HTTP request.
              -- Called on the request path.
              function envoy_on_request(request_handle)
                  local headers = request_handle:headers()
                  request_handle:streamInfo():dynamicMetadata():set("envoy.filters.http.lua", "request.info", {
                  host = headers:get(":authority"),
                  token = headers:get("x-api-key"),
                })
              end
              -- Called on the response path.
              function envoy_on_response(response_handle)
                local meta = response_handle:streamInfo():dynamicMetadata():get("envoy.filters.http.lua")["request.info"]
                response_handle:logErr("get req in response Auth: "..meta.host..", token: "..meta.token)
                local status = response_handle:headers():get(":status")
                response_handle:logErr("resp Status: "..status)
                if (meta.host ~= nil and meta.token ~= nil and status == "200")
                then
                    local headers, body = response_handle:httpCall(
                        "outbound|8081||httpbin.default.svc.cluster.local",
                        {
                          [":method"] = "POST",
                          [":path"] = "/anything",
                          [":authority"] = meta.host,
                          ["Content-Type"] = "application/json"
                        },
                        "{\"host\":\""..meta.host.."\",\"api_key\":\""..meta.token.."\"}",
                        5000
                      )

                    response_handle:logErr("body: "..body)

                end
              end
  workloadSelector:
    labels:
      istio: ingressgateway
```

