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
![](https://ws1.sinaimg.cn/large/006tNc79gy1fzs288x1spj31ey0s0qbk.jpg)


## 风险
![](https://ws4.sinaimg.cn/large/006tNc79gy1fzs2bmm3a0j31g00jawlw.jpg)

# 设备管理
![](https://ws1.sinaimg.cn/large/006tNc79gy1fzs2e98o25j31bg0u0dpl.jpg)

![](https://ws4.sinaimg.cn/large/006tNc79gy1fzs2eknkzxj31fu0lkjui.jpg)

# 通知栏

![](https://ws3.sinaimg.cn/large/006tNc79gy1fzs2gksmrjj31hg0p0tgi.jpg)

![](https://ws3.sinaimg.cn/large/006tNc79gy1fzs2gxsyjpj31gi0gijv0.jpg)


# allowBackup

`AndroidManifest.xml application  android:allowBackup="true"`

应用数据备份和恢复
![](https://ws1.sinaimg.cn/large/006tNc79gy1fzsbbo0bjjj31fy0sm7kf.jpg)

```
可以设置密码
adb backup -nosystem -f <xx.ab> <package name>
```

[abe](https://github.com/nelenkov/android-backup-extractor)解析.ab

ab 2 tar
`abe unpack <backup.ab> <backup.tar> [password]`

`adb restore <.ab>`