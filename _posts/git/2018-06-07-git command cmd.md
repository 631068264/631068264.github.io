---
layout:     post
rewards: false
title:      git 操作
categories:
    - git
---

# 删除远程分支
```shell
git push origin --delete <branchName>

git show <tag-name>
git tag                            # see tag lists
git push origin <tag-name>         # push a single tag
git push --tags                    # push all local tags
git push origin --delete tag <tagname>

git branch | grep -v "master" | xargs git branch -D # GIT本地删除除master以外所有分支
```

[如何在 Git 里撤销(几乎)任何操作](http://blog.jobbole.com/87700/)

```shell
新增
git remote add origin <address>


重设
git remote set-url origin <new-address>
```

重设commit
tar sha 的上一个
`git rebase -i <earlier SHA>`
要丢弃一个 commit，只要在编辑器里删除那一行就行了。
修改的 `pick` 替换为 `r`
合并 `pick` 替换为 `f` 或者 `s`