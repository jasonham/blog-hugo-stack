---
title: 构建Docker镜像，并上传到aliyun的容器镜像仓库
description: 构建Docker镜像，并上传到aliyun的容器镜像仓库
date: 2023-10-25T16:59:12+08:00
lastmod: 2023-10-25T16:59:12+08:00
slug: docker

tags:
 - Docker
categories:
 - Docker
# draft: true
---
# 操作备忘
## 根据Dockerfile生成原始镜像
``` linux
docker build -t taidii_staff_manage:[镜像版本号] .
```
## 查看当前主机上的镜像
```
docker images
```
## 登入到aliyun的镜像仓库
```
docker login --username=hanyi@1620343577597358 registry.cn-hangzhou.aliyuncs.com
```
## 给镜像改名
```
docker tag [ImageId] registry.cn-hangzhou.aliyuncs.com/taidii/api_staff_manage:[镜像版本号]
```
## 推送镜像到仓库
```
docker push registry.cn-hangzhou.aliyuncs.com/taidii/api_staff_manage:[镜像版本号]
```