---
title: hexo邂逅travis
date: 2019-03-15 20:31:32
thumbnail: ./assets/travis-hexo.png
tags: ['hexo', 'travis']
---

> 使用travis自动化部署Blog到GitHub Page

## 准备工作

+ Github - 托管Blog工程和Page静态网站
+ Travis - 自动部署blog到VPS

## 创建github仓库托管静态页面

这里仓库名一定要以github.io结尾。新建后看可以在`Seting`看到github page域名为[用户名].github.io结尾

## 生成token

> 这是为了授权给travis管理Blog仓库。

在github的设置界面找到，`Seting`>`Developer setting`>`Personal access tokens`>`Generate new tokens` 给token新起一个名字，选择repe权限，这里全选。然后生成后记得保存token，界面一旦推出就需要重新生成了。

## Travis配置

1. 使用github账户登录travis，在主界面选择托管Blog源码的仓库开启travis服务。
2. 打开`setting`，找到`General`设置将`Build only if .travis.yml is present`和`Build pushed branches` 复选，travis会在你push代码到github仓库后触发travis拉去代码进行自动构建。
3. 在`AutoCancellation`设置将两个选项复选。
4. **重点** 在`Environment Variables`变量设置，添加刚生成的`token`为`__TOKEN__`。***`display value in log`***不要复选，复选将这些信息暴露在日志上，日志是公开的。

### .travis.yml配置

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

通过上述设置每次更新Blog提交到github后travis会自动拉取构建执行`hexo deploy`后就会推送到仓库master分支。


