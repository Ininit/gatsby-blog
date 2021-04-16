---
title: hexo邂逅travis
date: 2019-03-15 20:31:32
thumbnail: ./assets/travis-hexo.png
tags: ['hexo', 'travis']
---

> 使用 travis 自动化部署 Blog 到 GitHub Page

## 准备工作

- Github - 托管 Blog 工程和 Page 静态网站
- Travis - 自动部署 blog 到 VPS

## 创建 github 仓库托管静态页面

这里仓库名一定要以 github.io 结尾。新建后看可以在`Seting`看到 github page 域名为[用户名].github.io 结尾

## 生成 token

> 这是为了授权给 travis 管理 Blog 仓库。

在 github 的设置界面找到，`Seting`>`Developer setting`>`Personal access tokens`>`Generate new tokens` 给 token 新起一个名字，选择 repe 权限，这里全选。然后生成后记得保存 token，界面一旦推出就需要重新生成了。

## Travis 配置

1. 使用 github 账户登录 travis，在主界面选择托管 Blog 源码的仓库开启 travis 服务。
2. 打开`setting`，找到`General`设置将`Build only if .travis.yml is present`和`Build pushed branches` 复选，travis 会在你 push 代码到 github 仓库后触发 travis 拉去代码进行自动构建。
3. 在`AutoCancellation`设置将两个选项复选。
4. **重点** 在`Environment Variables`变量设置，添加刚生成的`token`为`__TOKEN__`。**_`display value in log`_**不要复选，复选将这些信息暴露在日志上，日志是公开的。

### .travis.yml 配置

```
language: node_js
dist: trusty
node_js: stable
install:
- npm install hexo-cli -g
- npm install

before_script:
- git config --global user.name "ininit"
- git config --global user.email "meininit@gmail.com"
- sed -i "s/__TOKEN__/${__TOKEN__}/" _config.yml

script:
- hexo clean
- hexo generate
- hexo deploy
branches:
  only:
  - source
```

## 最后

通过上述设置每次更新 Blog 提交到 github 后 travis 会自动拉取构建执行`hexo deploy`后就会推送到仓库 master 分支。
