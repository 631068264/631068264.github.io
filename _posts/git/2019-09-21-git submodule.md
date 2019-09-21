---
layout:     post
rewards: false
title:      git 子模块
categories:
    - git
---

# 已存在子项目引入到not exist dir

```
git submodule add repo_url tar_dir
```

# 子项目加入到exist dir

```
git rm -r --cached tar_dir

git submodule add repo_url tar_dir
```

# 第一次clone含子模块的项目

clone main project 后子项目只有空目录

```
git submodule update --init
```

# 本地更新子项目  master分支

```
git submodule update --remote [子模块名]
```
