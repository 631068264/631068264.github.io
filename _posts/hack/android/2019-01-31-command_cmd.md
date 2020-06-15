---
layout:     post
rewards: false
title:     常用命令
categories:
    - hack
tags:
    - android hack
---

# 常用命令

- adb连接情况

```shell
adb devices -l

adb -s emulator-5556 shell

adb root
```

- 获取当前包名和入口类名

```shell
adb shell dumpsys window windows | grep -E 'mCurrentFocus'

adb shell dumpsys activity top|grep ACTIVITY
```

```
包名/入口类名
mCurrentFocus=Window{c61e50b u0 com.tencent.mm/com.tencent.mm.ui.LauncherUI}

```


- 包信息
AndroidManifest.xml

```
adb shell dumpsys package <包名>

aapt dump xmltree <apk> AndroidManifest.xml > <.txt>
```

- db信息

```
adb shell dumpsys dbinfo <包名>
```

- apk 文件

apk
```
adb install [-r 升级安装] /uninstall
```

get apk
```
# show all package
adb shell pm list packages

# get base apk
adb shell pm path <pageage>

adb pull <apk path> <local path>
```


文件
```
adb pull（拉到本地） / push（push到设备）  src tar
```

截图root
```
adb shell screencap -p <设备路径>

adb shell screenrecord <设备路径 sdcard/tmp.mp4>
adb pull sdcard/tmp.mp4 Downloads
adb shell rm -rf sdcard/tmp.mp4
```

- log

```
adb logcat -s tag
```
