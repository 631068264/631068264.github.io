---
layout:     post
rewards: false
title:      Install Parallel tools Kali 2019
categories:
    - hack
tags:
    - kali
---


# ssh

```
vim /etc/apt/sources.list

deb http://mirrors.aliyun.com/kali kali-rolling main non-free contrib 




systemctl start ssh.socket
systemctl enable ssh.socket

vim /etc/ssh/sshd_config

FROM:
#PermitRootLogin prohibit-password
TO:
PermitRootLogin yes
```




# fix

```
mkdir ~/tool
cp -rf /media/cdrom0 tool
cd ~/tool
chmod +wx * -R
```

run里面`./install`


```
cd kmods
tar zxvf prl_mod.tar.gz
```





# prl_eth/pvmnet/pvmnet.o

```
make[2]: Entering directory '/usr/src/linux-headers-5.3.0-kali2-amd64'
  CC [M]  /usr/lib/parallels-tools/kmods/prl_eth/pvmnet/pvmnet.o
/usr/lib/parallels-tools/kmods/prl_eth/pvmnet/pvmnet.c:194:3: error: 'struct ethtool_ops' has no member named 'get_settings'; did you mean 'get_strings'?
  194 |  .get_settings           = pvmnet_get_settings,
      |   ^~~~~~~~~~~~
      |   get_strings
/usr/lib/parallels-tools/kmods/prl_eth/pvmnet/pvmnet.c:194:28: error: initialization of 'void (*)(struct net_device *, struct ethtool_drvinfo *)' from incompatible pointer type 'int (*)(struct net_device *, struct ethtool_cmd *)' [-Werror=incompatible-pointer-types]
  194 |  .get_settings           = pvmnet_get_settings,
      |                            ^~~~~~~~~~~~~~~~~~~
/usr/lib/parallels-tools/kmods/prl_eth/pvmnet/pvmnet.c:194:28: note: (near initialization for 'pvmnet_ethtool_ops.get_drvinfo')
cc1: some warnings being treated as errors
make[4]: *** [/usr/src/linux-headers-5.3.0-kali2-common/scripts/Makefile.build:286: /usr/lib/parallels-tools/kmods/prl_eth/pvmnet/pvmnet.o] Error 1
make[3]: *** [/usr/src/linux-headers-5.3.0-kali2-common/Makefile:1639: _module_/usr/lib/parallels-tools/kmods/prl_eth/pvmnet] Error 2
make[2]: Leaving directory '/usr/src/linux-headers-5.3.0-kali2-amd64'
make[2]: *** [/usr/src/linux-headers-5.3.0-kali2-common/Makefile:179: sub-make] Error 2
make[1]: Leaving directory '/usr/lib/parallels-tools/kmods/prl_eth/pvmnet'
make[1]: *** [/usr/lib/parallels-tools/kmods/prl_eth/pvmnet/Makefile.v26:11: all] Error 2
make: Leaving directory '/usr/lib/parallels-tools/kmods'
make: *** [Makefile.kmods:49: compile] Error 2
Error: could not build kernel modules
Error during report about failed installation of parallels tools.
```




```
vim prl_eth/pvmnet/pvmnet.c


| and replace the line (or just delete the line)
|------------------------------------------------------------
| 	.get_settings = pvmnet_get_settings,
|------------------------------------------------------------
| with
|------------------------------------------------------------
| #ifndef ETHTOOL_GLINKSETTINGS
| 	.get_settings = pvmnet_get_settings,
| #endif
|------------------------------------------------------------

```

# prl_fs/super.c
```
/usr/lib/parallels-tools/kmods/prl_fs/SharedFolders/Guest/Linux/prl_fs/super.c: In function 'prlfs_remount':
/usr/lib/parallels-tools/kmods/prl_fs/SharedFolders/Guest/Linux/prl_fs/super.c:119:21: error: 'MS_RDONLY' undeclared (first use in this function); did you mean 'IS_RDONLY'?
  119 |  if ( (!((*flags) & MS_RDONLY) && PRLFS_SB(sb)->readonly) ||
      |                     ^~~~~~~~~
      |                     IS_RDONLY
/usr/lib/parallels-tools/kmods/prl_fs/SharedFolders/Guest/Linux/prl_fs/super.c:119:21: note: each undeclared identifier is reported only once for each function it appears in
/usr/lib/parallels-tools/kmods/prl_fs/SharedFolders/Guest/Linux/prl_fs/super.c:120:21: error: 'MS_MANDLOCK' undeclared (first use in this function); did you mean 'IS_MANDLOCK'?
  120 |         ((*flags) & MS_MANDLOCK) )
      |                     ^~~~~~~~~~~
      |                     IS_MANDLOCK
/usr/lib/parallels-tools/kmods/prl_fs/SharedFolders/Guest/Linux/prl_fs/super.c:123:12: error: 'MS_SYNCHRONOUS' undeclared (first use in this function); did you mean 'SB_SYNCHRONOUS'?
  123 |  *flags |= MS_SYNCHRONOUS; /* silently don't drop sync flag */
      |            ^~~~~~~~~~~~~~
      |            SB_SYNCHRONOUS
/usr/lib/parallels-tools/kmods/prl_fs/SharedFolders/Guest/Linux/prl_fs/super.c: In function 'prlfs_fill_super':
/usr/lib/parallels-tools/kmods/prl_fs/SharedFolders/Guest/Linux/prl_fs/super.c:273:17: error: 'MS_NOATIME' undeclared (first use in this function); did you mean 'S_NOATIME'?
  273 |  sb->s_flags |= MS_NOATIME | MS_SYNCHRONOUS;
      |                 ^~~~~~~~~~
      |                 S_NOATIME
/usr/lib/parallels-tools/kmods/prl_fs/SharedFolders/Guest/Linux/prl_fs/super.c:273:30: error: 'MS_SYNCHRONOUS' undeclared (first use in this function); did you mean 'SB_SYNCHRONOUS'?
  273 |  sb->s_flags |= MS_NOATIME | MS_SYNCHRONOUS;
      |                              ^~~~~~~~~~~~~~
      |                              SB_SYNCHRONOUS
make[4]: *** [/usr/src/linux-headers-5.3.0-kali2-common/scripts/Makefile.build:286: /usr/lib/parallels-tools/kmods/prl_fs/SharedFolders/Guest/Linux/prl_fs/super.o] Error 1
make[3]: *** [/usr/src/linux-headers-5.3.0-kali2-common/Makefile:1639: _module_/usr/lib/parallels-tools/kmods/prl_fs/SharedFolders/Guest/Linux/prl_fs] Error 2
make[2]: *** [/usr/src/linux-headers-5.3.0-kali2-common/Makefile:179: sub-make] Error 2
make[1]: *** [/usr/lib/parallels-tools/kmods/prl_fs/SharedFolders/Guest/Linux/prl_fs/Makefile.v26:13: all] Error 2
make: *** [Makefile.kmods:52: compile] Error 2
Error: could not build kernel modules
```

```
vim prl_fs/SharedFolders/Guest/Linux/prl_fs/super.c
```

just add
```
#include <uapi/linux/mount.h>
```


# tar back and install

```
tar zcvf prl_mod.tar.gz prl_* Makefile.kmods dkms.conf
```