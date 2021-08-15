---
layout: post
rewards: false
title:  apt离线包
categories:
    - Linux

---



# 离线包

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



# apt

不安装缓存到`/var/cache/apt/archives/*.deb` ，同时要减少来避免安装非必须的文件，从而减小包体积

- `-d` 仅下载 - 不安装或解压归档文件
- `-y` 对所有的询问选是
- `--no-install-recommends` 来避免安装非必须的文件



# pip

只下载不安装放到指定目录

- `download` 只下载不安装
- `-i` 指定源
- `-d` 缓存目录
- `-r` 依赖版本文件



# 安装

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