---
title: 初始化gitconfig
date: 2019-03-15 23:47:58
thumbnail: ./assets/gitconfig.png
tags: ['git']
---

## 初始化 gitconfig

> 新环境一切以 config 为头

1. 设置用户名 `git config global user.name "Ininit"`
2. 设置邮箱 `git config global user.email "explam@abc.com`
3. 设置 vsc 为默认编辑器 `git config global core.editor "code --wait"`
4. 设置 difftool `git config --global -e` 粘贴

```
[diff]
    tool = vscode
[difftool "vscode"]
    cmd = code --wait --diff $LOCAL $REMOTE
```

> 待续。。。
