---
layout:     post
rewards: false
title:      WordPress for mac
categories:
    - tools
---

# mamp

[mamp for mac](https://www.mamp.info/en/downloads/)
安装过程就是不停next,装完后非常蛋疼，<span class='heimu'> 明明我只想要mamp TMD 还装了mamp pro我有点洁癖，但是不敢动手干掉pro, </span> 因为php不懂

## 端口
设置一下 端口，由于本来就有mysql:3306 所以我选了default
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fxpu9anyedj315o0tiqgs.jpg)

## 建库
- start server 去到 `http//localhost:8888/phpMyAdmin`

- 或者到 `/Applications/MAMP/bin/phpMyAdmin/config.inc.php` 瞄一下配置才知道默认是这样的

```php
$cfg['Servers'][$i]['user']          = 'root';
$cfg['Servers'][$i]['password']      = 'root';
```

由于没用过phpMyAdmin就用客户端连上了。



# wordpress

[wordpress下载](https://wordpress.org/download/)

下载解压放到 htdocs 下面
![](https://ws3.sinaimg.cn/large/006tNbRwgy1fxq0a9dan8j30tu0oedk1.jpg)

访问 `http://localhost:8888/wordpress` 进行配置 （路径和解压后的文件名一致 可以改名）

按照提示填写 数据库 填上上面的建的库

# 设置中文
![](https://ws2.sinaimg.cn/large/006tNbRwgy1fxq1dalt76j31om0daq3f.jpg)