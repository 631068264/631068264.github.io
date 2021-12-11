---
layout:     post
rewards: false
title:      docker proxy 三种代理方式
categories:
    - docker

---

遇到问题docker pull 不走**~/.docker/config.json**代理的问题。

# dockerd代理 

[dokcker pull 代理](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy)

在执行`docker pull`时，是由守护进程`dockerd`来执行。 因此，代理需要配在`dockerd`的环境中。 而**这个环境，则是受systemd所管控，因此实际是systemd的配置。**

```bash
mkdir -p /etc/systemd/system/docker.service.d

vim /etc/systemd/system/docker.service.d/http-proxy.conf

[Service]
Environment="HTTP_PROXY=http://proxy.example.com:8080/"
Environment="HTTPS_PROXY=http://proxy.example.com:8080/"
Environment="NO_PROXY=localhost,127.0.0.1,.example.com"


systemctl daemon-reload
systemctl restart docker
```



# Container代理

在容器运行阶段，如果需要代理上网，则需要配置`~/.docker/config.json`。 [以下配置](https://docs.docker.com/network/proxy/)，只在Docker 17.07及以上版本生效。

```js
{
 "proxies":
 {
   "default":
   {
     "httpProxy": "http://proxy.example.com:8080",
     "httpsProxy": "http://proxy.example.com:8080",
     "noProxy": "localhost,127.0.0.1,.example.com"
   }
 }
}
```

此外，容器的网络代理，也可以直接在其运行时通过`-e`注入`http_proxy`等环境变量。 这两种方法分别适合不同场景。 `config.json`非常方便，默认在所有配置修改后启动的容器生效，适合个人开发环境。 在CI/CD的自动构建环境、或者实际上线运行的环境中，这种方法就不太合适，用`-e`注入这种显式配置会更好，减轻对构建、部署环境的依赖。 当然，在这些环境中，最好用良好的设计避免配置代理上网。

# docker build代理

虽然`docker build`的本质，也是启动一个容器，但是环境会略有不同，用户级配置无效。 在构建时，需要注入`http_proxy`等参数。

```sh
docker build . \
    --build-arg "HTTP_PROXY=http://proxy.example.com:8080/" \
    --build-arg "HTTPS_PROXY=http://proxy.example.com:8080/" \
    --build-arg "NO_PROXY=localhost,127.0.0.1,.example.com" \
    -t your/image:tag
```

**注意**：无论是`docker run`还是`docker build`，默认是网络隔绝的。 如果代理使用的是`localhost:3128`这类，则会无效。 这类仅限本地的代理，必须加上`--network host`才能正常使用。 而一般则需要配置代理的外部IP，而且代理本身要开启gateway模式。



# pull mirror host

**/etc/docker/daemon.json**

[daemon.json 配置官方详解](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)

**native.cgroupdriver=systemd**：kubelet 使用 `systemd` 作为 cgroup 驱动。 对于 Docker, 以此使系统更为稳定。

**insecure-registries**： 信任自签名证书，使用http（默认https）

**registry-mirrors**：设置默认仓库

**live-restore**：[在守护进程停机期间保持容器处于活动状态](

```json
{
	// 默认https 配置这个绕开
  "insecure-registries":["xxx:8080"],

  //自定义仓库
  "registry-mirrors": ["http://xxx:8080"]
}
```

