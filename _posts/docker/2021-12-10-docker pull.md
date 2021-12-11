---
layout:     post
rewards: false
title:      docker pull 过程
categories:
    - docker
---

[docker 源码分析(5) -- 镜像拉取及存储](https://yanhang.me/post/2015-03-30-docker-source-code-part5-images/)

# 大概流程

最近搞镜像仓库反向代理遇到的问题，代理8443，服务端是20000，因为实际上域名指向代理服务器，而服务器根本没有20000端口，所以理所当然refused。

```
Error response from daemon: Get http://reg.mydomain.com:8443/v2/nginx/manifests/test: Get http://reg.mydomain.com:20000/service/token?account=admin&scope=repository%3Anginx%3Apull&service=harbor-registry: dial tcp xxxx:20000: connect: connection refused
```

![ ](https://tva1.sinaimg.cn/large/008i3skNly1gx8tb8fwddj30bk09ft8x.jpg)

1. `docker client`从官方 Index(“index.docker.io/v1”)查询镜像(“samalba/busybox”)的 位置
2. Index 回复:
   - `samalba/busybox`在`Registry A`上
   - `samalba/busybox`的校验码
   - token
3. `docker client` 连接`Registry A`表示自己要获取`samalba/busybox`
4. `Registry A` 询问`Index` 这个客户端(`token/user`)是否有权限下载镜像
5. `Index`回复是否可以下载
6. 下载镜像的所有`layers`






```
docker  pull registry.aliyuncs.com/acs-sample/ubuntu
```

# 获取认证URL

```shell
curl -v "https://registry.aliyuncs.com/v2/"

< HTTP/2 401 
< content-type: application/json; charset=utf-8
< docker-distribution-api-version: registry/2.0
< www-authenticate: Bearer realm="https://dockerauth.aliyuncs.com/auth",service="registry.aliyuncs.com:cn-hangzhou:26842"
< content-length: 87
< date: Fri, 24 Aug 2018 02:47:18 GMT

```

可以看到在第二步这里卡住了。

# 获取token

```shell
curl -v "https://dockerauth.aliyuncs.com/auth?service=registry.aliyuncs.com%3Acn-hangzhou%3A26842&scope=repository%3Aacs-sample%2Fubuntu%3Apull"

{"access_token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjRSSU06SEhMNDpHU1MyOjdaQ0w6QkNMRDpKN0ZIOlVPNzM6Q1FETzpNUUg1OjdNQ1E6T0lQUTpYQlk1In0.eyJpc3MiOiJkb2NrZXJhdXRoLmFsaXl1bmNzLmNvbSIsImF1ZCI6InJlZ2lzdHJ5LmFsaXl1bmNzLmNvbTpjbi1oYW5nemhvdToyNjg0MiIsInN1YiI6IiIsImlhdCI6MTUzNTA3OTA3NiwianRpIjoiS1IwckZoMXM1YWM3S0VpMzEyeTJ2ZyIsIm5iZiI6MTUzNTA3ODc3NiwiZXhwIjoxNTM1MDc5Njc2LCJhY2Nlc3MiOlt7Im5hbWUiOiJhY3Mtc2FtcGxlL3VidW50dSIsInR5cGUiOiJyZXBvc2l0b3J5IiwiYWN0aW9ucyI6WyJwdWxsIl19XX0.OoIPkugzIpsdnxY2-qRgwwefAiB1A4gZQm_CJi97l33RDS81HnCn-OkqGvYPo03jbEF7iueAVBvcso8xvTUQFrIoEBVCuJuYv1mVh4_dNY4sjnxoUZvyHq8RoQ1w4ETLADoNf-k7HKCQs-PYPj7mmoBFxSBgpvG8VowUwc-oPbLp9cHe9_bE0gFvlSY7J5sv8egTUlrLzZWtVND7zyka2M3JxP70W4gFzt2-7XpsUqqmQqt6oS4o10_3b7-Vhah4XOqzN7t6g4PZ7LWu4yLWQmnRkH9baq1t53WbtexzTzWdYz5QXM9QglIx-yWwNxEJ6lbyv_wuduNBgLXQL5h8Eg","token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjRSSU06SEhMNDpHU1MyOjdaQ0w6QkNMRDpKN0ZIOlVPNzM6Q1FETzpNUUg1OjdNQ1E6T0lQUTpYQlk1In0.eyJpc3MiOiJkb2NrZXJhdXRoLmFsaXl1bmNzLmNvbSIsImF1ZCI6InJlZ2lzdHJ5LmFsaXl1bmNzLmNvbTpjbi1oYW5nemhvdToyNjg0MiIsInN1YiI6IiIsImlhdCI6MTUzNTA3OTA3NiwianRpIjoiS1IwckZoMXM1YWM3S0VpMzEyeTJ2ZyIsIm5iZiI6MTUzNTA3ODc3NiwiZXhwIjoxNTM1MDc5Njc2LCJhY2Nlc3MiOlt7Im5hbWUiOiJhY3Mtc2FtcGxlL3VidW50dSIsInR5cGUiOiJyZXBvc2l0b3J5IiwiYWN0aW9ucyI6WyJwdWxsIl19XX0.OoIPkugzIpsdnxY2-qRgwwefAiB1A4gZQm_CJi97l33RDS81HnCn-OkqGvYPo03jbEF7iueAVBvcso8xvTUQFrIoEBVCuJuYv1mVh4_dNY4sjnxoUZvyHq8RoQ1w4ETLADoNf-k7HKCQs-PYPj7mmoBFxSBgpvG8VowUwc-oPbLp9cHe9_bE0gFvlSY7J5sv8egTUlrLzZWtVND7zyka2M3JxP70W4gFzt2-7XpsUqqmQqt6oS4o10_3b7-Vhah4XOqzN7t6g4PZ7LWu4yLWQmnRkH9baq1t53WbtexzTzWdYz5QXM9QglIx-yWwNxEJ6lbyv_wuduNBgLXQL5h8Eg"}
```

# 通过token获取image下载配置

docker发送image的名称+tag（或者digest）给registry服务器，服务器根据收到的image的名称+tag（或者digest），找到相应image的manifest，然后将manifest返回给docker

```shell
curl -v -H "Accept: application/vnd.docker.distribution.manifest.v2+json" -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjRSSU06SEhMNDpHU1MyOjdaQ0w6QkNMRDpKN0ZIOlVPNzM6Q1FETzpNUUg1OjdNQ1E6T0lQUTpYQlk1In0.eyJpc3MiOiJkb2NrZXJhdXRoLmFsaXl1bmNzLmNvbSIsImF1ZCI6InJlZ2lzdHJ5LmFsaXl1bmNzLmNvbTpjbi1oYW5nemhvdToyNjg0MiIsInN1YiI6IiIsImlhdCI6MTUzNTA3OTA3NiwianRpIjoiS1IwckZoMXM1YWM3S0VpMzEyeTJ2ZyIsIm5iZiI6MTUzNTA3ODc3NiwiZXhwIjoxNTM1MDc5Njc2LCJhY2Nlc3MiOlt7Im5hbWUiOiJhY3Mtc2FtcGxlL3VidW50dSIsInR5cGUiOiJyZXBvc2l0b3J5IiwiYWN0aW9ucyI6WyJwdWxsIl19XX0.OoIPkugzIpsdnxY2-qRgwwefAiB1A4gZQm_CJi97l33RDS81HnCn-OkqGvYPo03jbEF7iueAVBvcso8xvTUQFrIoEBVCuJuYv1mVh4_dNY4sjnxoUZvyHq8RoQ1w4ETLADoNf-k7HKCQs-PYPj7mmoBFxSBgpvG8VowUwc-oPbLp9cHe9_bE0gFvlSY7J5sv8egTUlrLzZWtVND7zyka2M3JxP70W4gFzt2-7XpsUqqmQqt6oS4o10_3b7-Vhah4XOqzN7t6g4PZ7LWu4yLWQmnRkH9baq1t53WbtexzTzWdYz5QXM9QglIx-yWwNxEJ6lbyv_wuduNBgLXQL5h8Eg" "https://registry.aliyuncs.com/v2/acs-sample/ubuntu/manifests/latest"


{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/octet-stream",
      "size": 3820,
      "digest": "sha256:4791cda23dbc3d1c7a0491644ae1c819c7d24b516be95df79113119e0f073416"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 65699368,
         "digest": "sha256:56eb14001cebec19f2255d95e125c9f5199c9e1d97dd708e1f3ebda3d32e5da7"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 101415,
         "digest": "sha256:7ff49c327d838cf14f7db33fa44f6057b7209298e9c03369257485a085e231df"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 365,
         "digest": "sha256:6e532f87f96dd5821006d02e65e7d4729a4e6957a34c3f4ec72046e221eb7c52"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 681,
         "digest": "sha256:3ce63537e70c2c250fbc41b5f04bfb31f445be4034effc4b4c513bf8899dfa0a"
      },
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 686,
         "digest": "sha256:a521f68c7946d409bc182c615aa77d7e89de0914f67b2c0ce9be9f4c8e27c949"
      }
   ]
```



# 获取config

docker得到manifest后，读取里面image配置文件的digest(sha256)，这个sha256码就是image的ID。根据ID在本地找有没有存在同样ID的image，有的话就不用继续下载了



如果没有，那么会给registry服务器发请求（里面包含配置文件的sha256和media type），拿到image的配置文件（Image Config）



```shell
curl -v -H "Accept: application/vnd.docker.distribution.manifest.v2+json" -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IjRSSU06SEhMNDpHU1MyOjdaQ0w6QkNMRDpKN0ZIOlVPNzM6Q1FETzpNUUg1OjdNQ1E6T0lQUTpYQlk1In0.eyJpc3MiOiJkb2NrZXJhdXRoLmFsaXl1bmNzLmNvbSIsImF1ZCI6InJlZ2lzdHJ5LmFsaXl1bmNzLmNvbTpjbi1oYW5nemhvdToyNjg0MiIsInN1YiI6IiIsImlhdCI6MTUzNTA3OTA3NiwianRpIjoiS1IwckZoMXM1YWM3S0VpMzEyeTJ2ZyIsIm5iZiI6MTUzNTA3ODc3NiwiZXhwIjoxNTM1MDc5Njc2LCJhY2Nlc3MiOlt7Im5hbWUiOiJhY3Mtc2FtcGxlL3VidW50dSIsInR5cGUiOiJyZXBvc2l0b3J5IiwiYWN0aW9ucyI6WyJwdWxsIl19XX0.OoIPkugzIpsdnxY2-qRgwwefAiB1A4gZQm_CJi97l33RDS81HnCn-OkqGvYPo03jbEF7iueAVBvcso8xvTUQFrIoEBVCuJuYv1mVh4_dNY4sjnxoUZvyHq8RoQ1w4ETLADoNf-k7HKCQs-PYPj7mmoBFxSBgpvG8VowUwc-oPbLp9cHe9_bE0gFvlSY7J5sv8egTUlrLzZWtVND7zyka2M3JxP70W4gFzt2-7XpsUqqmQqt6oS4o10_3b7-Vhah4XOqzN7t6g4PZ7LWu4yLWQmnRkH9baq1t53WbtexzTzWdYz5QXM9QglIx-yWwNxEJ6lbyv_wuduNBgLXQL5h8Eg" "https://registry.aliyuncs.com/v2/acs-sample/ubuntu/blobs/sha256:4791cda23dbc3d1c7a0491644ae1c819c7d24b516be95df79113119e0f073416"

// 阿里云返回307

< HTTP/2 307 
< content-type: application/octet-stream
< docker-distribution-api-version: registry/2.0
< location: http://aliregistry.oss-cn-hangzhou.aliyuncs.com/docker/registry/v2/blobs/sha256/47/4791cda23dbc3d1c7a0491644ae1c819c7d24b516be95df79113119e0f073416/data?Expires=1535097803&OSSAccessKeyId=Ygxs2RciveEJoGFt&Random=6127a94e-d33c-4bdb-b165-5287f064392d&Signature=UypBLzJY%2FjIwG2hQox6r3mSAkvg%3D
< content-length: 338
< date: Fri, 24 Aug 2018 07:43:23 GMT
< 
<a href="http://aliregistry.oss-cn-hangzhou.aliyuncs.com/docker/registry/v2/blobs/sha256/47/4791cda23dbc3d1c7a0491644ae1c819c7d24b516be95df79113119e0f073416/data?Expires=1535097803&amp;OSSAccessKeyId=Ygxs2RciveEJoGFt&amp;Random=6127a94e-d33c-4bdb-b165-5287f064392d&amp;Signature=UypBLzJY%2FjIwG2hQox6r3mSAkvg%3D">Temporary Redirect</a>.
```

根据配置文件中的diff_ids（每个diffid对应一个layer tar包的sha256，tar包相当于layer的原始格式），在本地找对应的layer是否存在

如果layer不存在，则根据manifest里面layer的sha256和media type去服务器拿相应的layer（相当去拿压缩格式的包）。

拿到后进行解压，并检查解压后tar包的sha256能否和配置文件（Image Config）中的diff_id对的上，对不上说明有问题，下载失败






```shell
curl -v "http://aliregistry.oss-cn-hangzhou.aliyuncs.com/docker/registry/v2/blobs/sha256/47/4791cda23dbc3d1c7a0491644ae1c819c7d24b516be95df79113119e0f073416/data?Expires=1535097803&OSSAccessKeyId=Ygxs2RciveEJoGFt&Random=6127a94e-d33c-4bdb-b165-5287f064392d&Signature=UypBLzJY%2FjIwG2hQox6r3mSAkvg%3D"

{"architecture":"amd64","author":"Li Yi \u003cdenverdino@gmail.com\u003e","config":{"Hostname":"24dcaea7d349","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["/bin/bash"],"ArgsEscaped":true,"Image":"sha256:7f9b7ce7d8bb9abae9359dc4307cd3a6beec75cecce6cfb38e8af344e5a495ee","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":[],"Labels":{}},"container":"58991a81322eb76dd9c5265fe0180a623029c2ed25ee6c7453d79767d9618a33","container_config":{"Hostname":"24dcaea7d349","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],"Cmd":["/bin/sh","-c","sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/' /etc/apt/sources.list"],"ArgsEscaped":true,"Image":"sha256:7f9b7ce7d8bb9abae9359dc4307cd3a6beec75cecce6cfb38e8af344e5a495ee","Volumes":null,"WorkingDir":"","Entrypoint":null,"OnBuild":[],"Labels":{}},"created":"2016-07-13T15:07:21.439478488Z","docker_version":"1.10.3","history":[{"created":"2016-06-24T17:29:06.589214339Z","created_by":"/bin/sh -c #(nop) ADD file:b6ff401cf2a7a08c11d2bdfbfec31c7ec105fd7ab29c529fb90025762b077e2c in /"},{"created":"2016-06-24T17:29:10.38986507Z","created_by":"/bin/sh -c set -xe \t\t\u0026\u0026 echo '#!/bin/sh' \u003e /usr/sbin/policy-rc.d \t\u0026\u0026 echo 'exit 101' \u003e\u003e /usr/sbin/policy-rc.d \t\u0026\u0026 chmod +x /usr/sbin/policy-rc.d \t\t\u0026\u0026 dpkg-divert --local --rename --add /sbin/initctl \t\u0026\u0026 cp -a /usr/sbin/policy-rc.d /sbin/initctl \t\u0026\u0026 sed -i 's/^exit.*/exit 0/' /sbin/initctl \t\t\u0026\u0026 echo 'force-unsafe-io' \u003e /etc/dpkg/dpkg.cfg.d/docker-apt-speedup \t\t\u0026\u0026 echo 'DPkg::Post-Invoke { \"rm -f /var/cache/apt/archives/*.deb /var/cache/apt/archives/partial/*.deb /var/cache/apt/*.bin || true\"; };' \u003e /etc/apt/apt.conf.d/docker-clean \t\u0026\u0026 echo 'APT::Update::Post-Invoke { \"rm -f /var/cache/apt/archives/*.deb /var/cache/apt/archives/partial/*.deb /var/cache/apt/*.bin || true\"; };' \u003e\u003e /etc/apt/apt.conf.d/docker-clean \t\u0026\u0026 echo 'Dir::Cache::pkgcache \"\"; Dir::Cache::srcpkgcache \"\";' \u003e\u003e /etc/apt/apt.conf.d/docker-clean \t\t\u0026\u0026 echo 'Acquire::Languages \"none\";' \u003e /etc/apt/apt.conf.d/docker-no-languages \t\t\u0026\u0026 echo 'Acquire::GzipIndexes \"true\"; Acquire::CompressionTypes::Order:: \"gz\";' \u003e /etc/apt/apt.conf.d/docker-gzip-indexes"},{"created":"2016-06-24T17:29:11.953495224Z","created_by":"/bin/sh -c rm -rf /var/lib/apt/lists/*"},{"created":"2016-06-24T17:29:13.569514251Z","created_by":"/bin/sh -c sed -i 's/^#\\s*\\(deb.*universe\\)$/\\1/g' /etc/apt/sources.list"},{"created":"2016-06-24T17:29:14.1074651Z","created_by":"/bin/sh -c #(nop) CMD [\"/bin/bash\"]","empty_layer":true},{"created":"2016-07-13T15:07:18.742108701Z","author":"Li Yi \u003cConnection #0 to host aliregistry.oss-cn-hangzhou.aliyuncs.com left intact
* Closing connection #0
denverdino@gmail.com\u003e","created_by":"/bin/sh -c #(nop) MAINTAINER Li Yi \u003cdenverdino@gmail.com\u003e","empty_layer":true},{"created":"2016-07-13T15:07:21.439478488Z","author":"Li Yi \u003cdenverdino@gmail.com\u003e","created_by":"/bin/sh -c sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/' /etc/apt/sources.list"}],"os":"linux","rootfs":{"type":"layers","diff_ids":["sha256:81a9ec52d927ef3bf2d3959adbb104cdb6b0a925e7f1587579501bb3c35ace2f","sha256:084c7f432685199a281b0c5bb99dd24edfbec2ec2c660c8d43ce1dbe774bf842","sha256:355edbeff03366b5092db6a663c793cd518ff359242e3618429152e8e8537758","sha256:69be5dd4a9a933deaae4147a8a704839350bbe0a3a624eb7f5de522337d332b6","sha256:d074943d867a103265390d0c3fd38907588602342855920883ae53d8a4e26a6b"]}}
```

