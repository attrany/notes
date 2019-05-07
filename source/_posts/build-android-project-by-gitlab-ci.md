---
title: 使用gitlab-ci进行android项目构建
date: 2017-03-02 14:53:37
tags: [gitlab, ci, gitlab-ci, android, build, 构建, 打包]
category: 工作日常
featured_image: https://attrany.oss-cn-beijing.aliyuncs.com/notes/20190507215118.png
---

## 背景
最近接了一个android内部应用的活，但是每次给别人试用的时候都是在本地打个包，然后用邮件或者IM传给大家。
效率低下，而且没有版本追踪。于是想到了CI。
之前用CI仅仅是跑一些自动测试，没有进行过android项目的打包发布的经历，不过感觉应该不难。
大致思路为：
代码合并至RELEASE分支 -> 触发一个WebHook通知CI构建 -> CI构建并完成打包 -> 发布程序
手头没有jenkins，而且刚好之前看到gitlab集成了ci功能，就想尝尝鲜。

## 配置构建环境
经过研究，gitlab-ci采用分布式构建模式，也就是构建过程是跑在不同的构建主机上的。
构建主机需要运行gitlab-ci-runner进程，与gitlab服务端进行通信，以接收并执行分派给该进程的构建任务。
跑构建的主机上可以部署gitlab-ci-runner程序，使用gitlab项目中配置的token启动以后，该runner会自动出现在gitlab服务端的项目视图中。
进入gitlab项目页面->设置下拉列表->runners即可看到

我厂的gitlab已经配置好了2个

## 创建构建配置文件
gitlab-ci的构建过程是在配置文件中配置合描述的。
那么接下来，在项目根目录创建.gitlab-ci.yml文件
配置文件可以包含如下几个段落
variables
before_script
stages
build
我们的项目需要进行如下步骤
编译打包 -> 发布程序
所以我们在build下面配置如下步骤
```
build:
  stage: build
  script:
    - ./gradlew assembleRelease
    - DATE=`date +%Y%m%d%H%M%S`
    - rsync ./app/build/outputs/apk/app-release.apk work@publish-server:/home/work/data/www/publish-site/app/mtaa/mtaa_${DATE}.apk
    - ssh work@publish-server "cd /home/work/data/www/publish-site/app/mtaa; cp -f mtaa_${DATE}.apk mtaa.apk"

  artifacts:
    paths:
    - app/build/outputs/
```

## 效果
构建过程
构建结果
发布结果
大家终于可以扫码下载最新版应用啦