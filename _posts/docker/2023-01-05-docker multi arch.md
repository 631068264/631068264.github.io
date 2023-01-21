---
layout:     post
rewards: false
title:     docker multi arch support
categories:
    - docker


---



# build

使用 [buildx](https://docs.docker.com/build/building/multi-platform/)

示例

```sh
docker buildx build --platform linux/arm64,linux/arm64,linux/arm --no-cache -f arm.Dockerfile -t image:tag .
```

macOS 或 Windows 系统的 Docker Desktop，以及 Linux 发行版通过 `deb` 或者 `rpm` 包所安装的 `docker` 内置了 `buildx`，不需要另行安装。 [官方安装教程](https://docs.docker.com/build/install-buildx/)

如果你的 `docker` 没有 `buildx` 命令，可以下载二进制包进行安装：

- 首先从 [Docker buildx](https://github.com/docker/buildx/releases/latest) 项目的 release 页面找到适合自己平台的二进制文件。

- 下载二进制文件到本地并重命名为 `docker-buildx`，移动到 docker 的插件目录 `~/.docker/cli-plugins`。

- 向二进制文件授予可执行权限。



# pull

[pull](https://docs.docker.com/engine/reference/commandline/image_pull/) 

```sh
docker pull --platform=arm64
```

使用`platform`参数可以跨架构可以pull镜像，docker默认下载适配架构的镜像，没有再下载其他的（例如只有amd64的，即使用了参数，也只能下载amd64）





# push

[mainfest](https://docs.docker.com/engine/reference/commandline/manifest/)

让多arch镜像使用同一个tag。

单个清单是有关图像的信息，例如层、大小和摘要。docker manifest 命令还为用户提供额外的信息，例如构建映像所针对的操作系统和体系结构。

清单列表是通过指定一个或多个（最好是多个）图像名称创建的图像层列表。例如，它可以像图像名称一样在命令中使用`docker pull`。`docker run`



**创建并推送清单列表**

https 自签名的话使用`--insecure`，`-p, --purge-镜像 push 成功后删除本地的多架构镜像mainfest`

```sh
docker manifest create --insecure myprivateregistry.mycompany.com/repo/image:1.0 \
    myprivateregistry.mycompany.com/repo/image-linux-ppc64le:1.0 \
    myprivateregistry.mycompany.com/repo/image-linux-s390x:1.0 \
    myprivateregistry.mycompany.com/repo/image-linux-arm:1.0 \
    myprivateregistry.mycompany.com/repo/image-linux-armhf:1.0 \
    myprivateregistry.mycompany.com/repo/image-windows-amd64:1.0 \
    myprivateregistry.mycompany.com/repo/image-linux-amd64:1.0
    

docker manifest push -p --insecure myprivateregistry.mycompany.com/repo/image:tag
```

- 为不同arch的镜像**本地**创建清单（**指向同一个image:tag**  例如myprivateregistry.mycompany.com/repo/image:1.0），而且其他arch镜像也得预先push，不能只是本地镜像，不然创建失败。
- push 实际上只是把清单内容push上去
- 可以通过`docker pull --platform=arm64`试验成果。







