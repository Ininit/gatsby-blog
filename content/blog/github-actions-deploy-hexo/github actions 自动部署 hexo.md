---
title: github actions 自动部署 hexo
date: 2020-04-29 23:47:58
thumbnail: ./assets/actions.png
tags: ['hexo']
---

## 准备

1. Personal access token - 用于 hexo deploy 部署到 github 的认证。设置在`setting/developer settings` 下
2. Secrets - 用户挂在在 github actions 下的 env 环境变量, 这里可以存放不想暴露在 actions log 中的字段, 比如上面的 access token 需要保存在此。设置在**repository**下的 setting

## hexo 配置改造

修改 \_config.yml 下 deploy 中的 repo 链接, `TOKEN` 就是我们刚生成的 access token, 这里使用 TOKEN 作为标记符号是为了在 actions 环境中方便替换。

```yml
deploy:
  type: git
  repo: https://TOKEN@github.com/Ininit/ininit.github.io
  branch: master
```

## workflows 文件配置

配置 github acitons 的配置文件, 在仓库`actions`中选择 nodejs 作为 workflow 环境

```yaml
# This workflow will do a clean install of node dependencies, build the source code and run tests across different versions of node
# For more information see: https://help.github.com/actions/language-and-framework-guides/using-nodejs-with-github-actions

name: deploy blog

on:
  push:
    branches: [source]

jobs:
  build:
    runs-on: macOS-latest ## 选择系统环境
    steps:
      - uses: actions/checkout@v1 ## 检出 github 代码

      - name: use nodejs
        uses: actions/setup-node@v1 ## 使用 node
        with:
          node-version: '12.x'

      - name: init
        run: |
          git config --global user.email "meininit@gmail.com"
          git config --global user.name "Ininit"
          npm install -g yarn
          yarn global add hexo-cli 
          yarn add -D hexo-deployer-git ## 使用 hexo deploy 中的 git 需要安装这个插件
      - name: yarn install
        run: |
          yarn

      - name: deploy
        env:
          TOKEN: ${{secrets.DEPLOY_KEY}} ## 这里就是在 Secrets 配置的 access token
        run: |
          sed -i "" "s/TOKEN/$TOKEN/" _config.yml ## 替换掉配置中的 TOKEN 标记
          yarn build
          echo "blog.ininit.me" >> public/CNAME ## 自定义域名配置  CNAME
          yarn deploy
```

## 总结

1. 我的是 源码与 page 是同一个仓库, 所以 branches 是 source 分支, page 在 master 分支
2. 关键在于设置 access token 到仓库的 secrets 中
