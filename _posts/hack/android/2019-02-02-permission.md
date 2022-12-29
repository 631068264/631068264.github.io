---
layout:     post
rewards: false
title:     permission 权限
categories:
    - hack
tags:
    - android hack
---

# 辅助功能
监听当前window变化
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030457.jpg)


## 风险
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030458.jpg)

# 设备管理
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030459.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030460.jpg)

# 通知栏

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030461.jpg)

![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030462.jpg)


# allowBackup

`AndroidManifest.xml application  android:allowBackup="true"`

应用数据备份和恢复
![](https://cdn.jsdelivr.net/gh/631068264/img/202212301030463.jpg)

```
可以设置密码
adb backup -nosystem -f <xx.ab> <package name>
```

[abe](https://github.com/nelenkov/android-backup-extractor)解析.ab

ab 2 tar
`abe unpack <backup.ab> <backup.tar> [password]`

`adb restore <.ab>`