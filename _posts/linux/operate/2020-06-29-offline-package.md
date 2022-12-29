---
layout: post
rewards: false
title:  ubuntu centos离线包
categories:
    - Linux

---



# ubuntu

## 离线包

```dockerfile
FROM xxxx

COPY sources.list /etc/apt/sources.list

RUN rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/* && apt update -y && \
	apt -d install -y --no-install-recommends build-essential python3.7 python3.7-dev python3-distutils libmysqlclient-dev libyaml-dev libffi-dev libssl-dev libbz2-dev liblz4-dev zlib1g-dev && \
    mkdir deb && cp -r /var/cache/apt/archives/*.deb deb

# prepare whl
RUN apt install -y --no-install-recommends wget build-essential python3.7 python3.7-dev python3-distutils libmysqlclient-dev libyaml-dev libffi-dev libssl-dev libbz2-dev liblz4-dev zlib1g-dev
RUN wget --no-check-certificate https://bootstrap.pypa.io/get-pip.py && python3.7 get-pip.py
COPY offline_req.txt .
RUN pip download -i https://mirrors.aliyun.com/pypi/simple/ -d wheelhouse -r offline_req.txt
```

## apt

不安装缓存到`/var/cache/apt/archives/*.deb` ，同时要减少来避免安装非必须的文件，从而减小包体积

- `-d` 仅下载 - 不安装或解压归档文件
- `-y` 对所有的询问选是
- `--no-install-recommends` 来避免安装非必须的文件

## pip

只下载不安装放到指定目录

- `download` 只下载不安装
- `-i` 指定源
- `-d` 缓存目录
- `-r` 依赖版本文件

## 安装

apt install

```shell
# 安装时不安装顺序会缺依赖，所以装两次
dpkg -i ${DEB_DIR}/*.deb || dpkg -i ${DEB_DIR}/*.deb
```



[pip install](https://pip.pypa.io/en/stable/reference/pip_install/)     [virtualenv](https://python-guide-kr.readthedocs.io/ko/latest/dev/virtualenvs.html)

```shell
# 装virtualenv包
pip install --no-cache-dir --no-index --find-links=${WHL_DIR} virtualenv
virtualenv -p python3.7 --no-site-packages --never-download ${LICENSE_VENV}

# 安装依赖
source ${LICENSE_VENV}/bin/activate
pip install --no-cache-dir --no-index --find-links=${WHL_DIR} -r requirements.txt
deactivate
```

- `--no-cache-dir` 禁用缓存 (**docker时候有效减少image体积**)  发出任何HTTP请求时，pip首先会检查其本地缓存，以确定是否为该请求存储了合适的响应且尚未过期。如果是这样，则仅返回该响应，而不发出请求

- `--no-index --find-links` 不从远程访问 , 使用本地目录

- `--no-site-packages` 将不包括已经全局安装的软件包 **default version>=1.7**
- `--never-download` 避免连网 **default version>=1.10**



> 还是docker方便

# Centos

[全量依赖 rpm 包及离线安装](https://cloud.tencent.com/developer/article/1614031)

```dockerfile
FROM centos:7
ARG PKG_NAME

COPY Centos-7.repo /etc/yum.repos.d/CentOS-Base.repo
RUN yum clean all \
    && yum makecache \
    && yum install yum-utils

#RUN mkdir -p /data/rpm  && repotrack --download_path=/data/rpm ${PKG_NAME}
RUN mkdir -p /data/rpm  && yumdownloader --resolve --destdir=/data/rpm ${PKG_NAME}
```

打包安装包

```sh
#!/bin/bash
IMAGE_NAME=centos-offline
CONTAINER_NAME=centos-offline
LOCAL_PATH=$1-rpm

set -ex

docker build --no-cache \
    --build-arg PKG_NAME=$1 \
    -t ${IMAGE_NAME} -f centos.Dockerfile .
mkdir -p ${LOCAL_PATH} && rm -rf ${LOCAL_PATH}
docker run -d --rm --name ${CONTAINER_NAME} ${IMAGE_NAME} sleep 1m
docker cp ${CONTAINER_NAME}:/data/rpm/ ${LOCAL_PATH}
docker kill ${CONTAINER_NAME}
tar -zcvf $1-rpm.tar.gz ${LOCAL_PATH} && rm -rf ${LOCAL_PATH}
```

安装  [rpm 命令](https://www.linuxcool.com/rpm)

```sh
tar -xzvf xxx.tar.gz -C path

# --force 忽略报错，强制安装  --nodeps就是安装时不检查依赖关系
rpm -Uvh *.rpm --nodeps --force
# 直接安装软件包
rpm -ivh --force *.rpm


# 解压rpm到指定目录，通过yum安装
cd $DECOMPRESS_PATH
createrepo ./
echo -e "[docker-ce]
name=docker-ce
baseurl=file://$DECOMPRESS_PATH
enabled=1
gpgcheck=0" > /etc/yum.repos.d/docker-ce.repo
# 禁用其他repo 更新repo
yum -y --disablerepo="*" --enablerepo="docker-ce" update
yum -y install docker-ce --disablerepo="*" --enablerepo=docker-ce
systemctl restart docker && systemctl enable docker





```



```sh
# Duplicate Package
package-cleanup --dupes # lists duplicate packages
package-cleanup --cleandupes # removes duplicate packages



yum list installed > my_list.txt
yum list installed | grep nginx
```



