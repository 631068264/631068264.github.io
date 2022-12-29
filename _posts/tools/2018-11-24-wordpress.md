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
![](https://cdn.jsdelivr.net/gh/631068264/img/006tNbRwgy1fxpu9anyedj315o0tiqgs.jpg)

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
![](https://cdn.jsdelivr.net/gh/631068264/img/006tNbRwgy1fxq0a9dan8j30tu0oedk1.jpg)

访问 `http://localhost:8888/wordpress` 进行配置 （路径和解压后的文件名一致 可以改名）

按照提示填写 数据库 填上上面的建的库

# 设置中文
![](https://cdn.jsdelivr.net/gh/631068264/img/006tNbRwgy1fxq1dalt76j31om0daq3f.jpg)

# 设置API
- [WP API doc](https://developer.wordpress.org/rest-api/reference/)
- [JWT Authentication](https://wordpress.org/plugins/jwt-authentication-for-wp-rest-api/)

配置 .htaccess
```shell
find / -name .htaccess

# 添加在配置最前面 不然会报错
RewriteEngine on
RewriteCond %{HTTP:Authorization} ^(.*)
RewriteRule ^(.*) - [E=HTTP_AUTHORIZATION:%1]
SetEnvIf Authorization "(.*)" HTTP_AUTHORIZATION=$1
```
**不要配置在里面**不然会因为某些原因重写.htaccess被覆盖
```
# BEGIN WordPress
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /wordpress/
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /wordpress/index.php [L]
</IfModule>

# END WordPress
```


配置 wp-config.php 密钥 https://api.wordpress.org/secret-key/1.1/salt/
```shell
/** 设置WordPress变量和包含文件。 */
define('JWT_AUTH_SECRET_KEY', '');
define('JWT_AUTH_CORS_ENABLE', true);
...

```


# ACF API

[demo](https://github.com/airesvsg/acf-to-rest-api-example)

自定义字段需要`fields[xx]`post请求。编辑字段需要先官方API发表文章获取id,再通过ACF api 来修改acf字段
