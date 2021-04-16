---
title: sentry之docker安装与配置
thumbnail: ./assets/sentry.jpeg
date: 2019-07-11 20:56:19
tags: ['docker', 'sentry']
---

## 前言

最近公司再搞重构，我顺势也把之前遗留问题解决并加上 bug 收集机制。可以及时看到用户在自己项目使用中产生的 bug，顿时神清气爽。

sentry 是一个支持大多数语言和框架 bug 收集报告的平台， 该平台有开源和 saas 服务，这里我们使用它开源代码进行自建 bug 收集平台。

## 准备工作

> 我是给公司配置该服务，直接用公司服务器线上服务器。

1. 一台 Linux （这里我直接用 docker 进行搭建，所以任意类型的服务器都问题不大）
2. Docker sentry 项目[onpremise](https://github.com/getsentry/onpremise)

## 安装

1. 拉去 onpremise 项目

```
git clone https://github.com/getsentry/onpremise.git
```

2. 安装

```
./install.sh
```

脚本执行过程中需要拉去 docker 镜像，速度慢的同学可以`google docker 镜像加速 网易`。 最后根据提示创建用户， 如果这过程没有看到，待完成自己执行`docker-compose run --rm web createuser`创建用户，这里我创建第一个用户设为超级管理员。

3. 完成

关闭该网站，打开官方文档，开始体验，就是这么简单。

## 配置 smtp

> 服务搭都搭建了，怎么能少了服务通知呢，我们不可能一直打开服务看着。我这里配置是使用 qq 邮箱 SMTP 进行邮件发送。我看大家基本都是搭建邮件转发钉钉，大家可以了解一下。

```
The recommended way to customize your configuration is using the files below, in that order:
  config.yml
  sentry.conf.py
  .env w/ environment variables
```

项目介绍提示我们配置按照三个配置文件优先级进行配置，优先使用 `config.yml`

```
mail.backend: 'smtp'
mail.host: 'smtp.qq.com'
mail.port: 587
mail.username: '10086@qq.com'
mail.password: '授权码'
mail.use-tls: true
mail.from: '10086@qq.com' # 这个一定要跟username一样不然qq会拒绝
```

完成这个前，要先到 `QQ邮箱/设置/账户` 开启 SMTP 服务。配置好后执行`docker-compose up`, 这里加上`-d`是为了方便观察问题，如果直接后台运行后出错是很难纠错，我们通过看 docker-compose.yml 文件可以看到每个服务`restart: unless-stoped`。如果不停止，他会一直尝试重启。当我们发现出错可以按`ctrl+c`停止容器运行。

## 独立服务

> 因为公司不可能只有 sentry 服务运行，还有其他服务，这时候我们可以通过定制把相同的服务进行合并，这里我将`redis`服务独立出来，再进行`docker-compose down/up`不会影响其他服务。

1. 注释掉 `sentry` 的 `redis` 服务

```docker-compose.yml
...
  depends_on:
#   - redis
    - postgres
    - memcached
    - smtp
...

...
#  redis:
#    restart: unless-stopped
#    image: redis:3.2-alpine
...
```

这样我们就将内置的 `redis` 去掉了，下面添加外部 `reids`

```docker-compose.yml
...
  environment:
    SENTRY_REDIS_HOST: redis # 这里不需要注释
...
```

如果直接在上面写外部 `redis` 名称是没办法访问到的，通过网上一顿瞎找， 找到可以将容器网络模式设为 `bridge` 桥接。

```docker-compose.yml
...
  volumes:
    - sentry-data:/var/lib/sentry/files
  network_mode: bridge
...
```

我们设置这里是不够的，`SMTP`，`memcached`，`postgres` 也需要跟 `web`, `worker`, `cron` 通信，所以我们也要将他们的网络模式设置为 `bridge`。

通过 `docker-compose up --build` 可以启动前重新构建，得益于 docker 缓存技术，几乎跟直接启动无差别。

启动后我们发现还是无法通信到外部 `redis`

通过 google 一顿查询，我们需要通过 link 机制将外部 `redis` 链接到 `web` 等服务

```docker-compose.yml
...
  network_mode: bridge
  external_links:
    - redis-server: redis # 这里用了别名，因为外部服务名字不叫redis，这里我就将它设置别名方便。
...
```

这种模式是单向的，就是 `sentry` 服务能够访问外部 `redis` 服务， 外部不能访问到 `sentry` 的服务。

一顿操作 `docker-compose up --build -d` 大功告成。

## 感谢

[Docker Compose：链接外部容器的几种方式](https://notes.doublemine.me/2017-06-12-Docker-Compose-%E9%93%BE%E6%8E%A5%E5%A4%96%E9%83%A8%E5%AE%B9%E5%99%A8%E7%9A%84%E5%87%A0%E7%A7%8D%E6%96%B9%E5%BC%8F.html)
