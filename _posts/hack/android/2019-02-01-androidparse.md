---
layout:     post
rewards: false
title:     android file parse
categories:
    - hack
tags:
    - android hack
---
# so file

## ELF
ELF(Executable and Linking Format)是一种对象文件的格式

用于定义不同类型的对象文件(Object files)中都放了什么东西、以及都以什么样的格式去放这些东西

- 可重定位的对象文件(Relocatable file)
由汇编器汇编生成的 .o 文件
- 可执行的对象文件(Executable file)
可执行应用程序
- 可被共享的对象文件(Shared object file)
动态库文件，也即 .so 文件

## 参数
so 头
```
readelf -h xx.so
```

- -a --all              全部       Equivalent to: -h -l -S -s -r -d -V -A -I
- -h --file-header      文件头   Display the ELF file header
- -l --program-headers  程序 Display the program headers
- --segments An alias for --program-headers
- -S --section-headers  段头 Display the sections' header
- --sections            An alias for --section-headers
- -e --headers          全部头      Equivalent to: -h -l -S
- -s --syms             符号表      Display the symbol table
- --symbols             An alias for --syms
- -n --notes            内核注释     Display the core notes (if present)
- -r --relocs           重定位     Display the relocations (if present)
- -u --unwind            Display the unwind info (if present)
- -d --dynamic          动态段     Display the dynamic segment (if present)
- -V --version-info     版本    Display the version sections (if present)
- -A --arch-specific    CPU构架   Display architecture specific information (if any).
- -D --use-dynamic      动态段    Use the dynamic section info when displaying symbols
- -x --hex-dump=<number> 显示 段内内容Dump the contents of section <number>
- -w[liaprmfFso] or
- -I --histogram         Display histogram of bucket list lengths
- -W --wide              宽行输出      Allow output width to exceed 80 characters
- -H --help              Display this information
- -v --version           Display the version number of readelf

![](https://ws1.sinaimg.cn/large/006tNc79gy1fztjvqtzqej316y0dm0x4.jpg)

- Seation Header
![](https://ws3.sinaimg.cn/large/006tNc79gy1fztjxxinrcj318e0k441k.jpg)

Seation Header一般都是多个用一个List来保存

- Program Headers
![](https://ws2.sinaimg.cn/large/006tNc79gy1fztk06dm8xj31380u07a6.jpg)


# Android manifest
## R id
![](https://ws2.sinaimg.cn/large/006tNc79gy1fzqwl7f94tj31du0u0kct.jpg)

## aapt
将Android打包成resource.arsc 从文本xml -> 二进制xml
![](https://ws4.sinaimg.cn/large/006tNc79gy1fzqxi7p2y0j31kq0q078s.jpg)
![](https://ws4.sinaimg.cn/large/006tNc79gy1fzqxqpq5i6j31pk0mmnb3.jpg)

在`[sdk home]/build-tools`
```
aapt l -a <apk> > xx.xml
```


## apk 构建过程
![](https://ws2.sinaimg.cn/large/006tNc79gy1fzqxvc7vzsj31800s2agi.jpg)


# Resource.arsc
![](https://ws2.sinaimg.cn/large/006tNc79gy1fzqyr6wl9aj31610u04gt.jpg)

# dex
在`[sdk home]/build-tools`

编译出dex
```
dx -dex --output=<path> <.class>
```