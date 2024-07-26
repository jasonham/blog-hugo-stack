---
title: 用 aws cli 新建 lightsail 实例
description: 因网络原因，无法使用 AWS 管理台新建 lightsail 实例，所以使用 aws cli 创建 lightsail 实例
date: 2024-07-26T13:24:03+08:00
lastmod: 2024-07-26T13:24:03+08:00
slug: create-new-lightsail-instance-using-aws-cli

tags:
  - aws
  - lightsail
categories:
  - 运维
draft: true
---

# 用 aws cli 新建 lightsail 实例

## 获取蓝图与样例的 ID

```bash
aws lightsail get-blueprints
aws lightsail get-bundles
```

## 删除旧实例

```bash
aws lightsail delete-instance --instance-name ubuntu
```

## 创建实例，替换实例名

```bash
aws lightsail create-instances \
 --instance-names "ubuntu" \
 --availability-zone ap-northeast-1a \
 --blueprint-id ubuntu_22_04 \
 --bundle-id nano_3_0 \
 --key-pair-name lightsail
```

## 获取实例公共 IP

```bash
aws lightsail get-instance --instance-name ubuntu | jq -r .instance.publicIpAddress
```

## 连接到实例

```bash
ssh -i ~/.ssh/lightsail.pem ubuntu@13.231.100.164
```

## 安装 outline

```bash
sudo bash -c "$(wget -qO- https://raw.githubusercontent.com/Jigsaw-Code/outline-server/master/src/server_manager/install_scripts/install_server.sh)"
```

## 根据 outline 提示开放端口

```bash
aws lightsail open-instance-public-ports \
 --instance-name ubuntu \
 --port-info fromPort=19688,toPort=19688,protocol=TCP \
&& \
aws lightsail open-instance-public-ports \
 --instance-name ubuntu \
 --port-info fromPort=31305,toPort=31305,protocol=TCP \
&& \
aws lightsail open-instance-public-ports \
 --instance-name ubuntu \
 --port-info fromPort=31305,toPort=31305,protocol=UDP
```
