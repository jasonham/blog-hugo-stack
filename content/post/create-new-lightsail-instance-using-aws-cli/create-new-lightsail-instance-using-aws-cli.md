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
---

# 用 aws cli 新建 lightsail 实例

## 获取蓝图与样例的 ID

```bash
aws lightsail get-blueprints
aws lightsail get-bundles
```

# 使用组合命令创建实例

```bash
# 定义变量
# 实例名称
instanceName="ubuntu"
# 选择需要的镜像ID
blueprintId="ubuntu_22_04"
# 选择需要的实例套餐ID
bundleId="nano_3_0"
# 选择需要的密钥对名称，需要提前创建到 aws 的密钥对管理中
keyPairName="lightsail"

# 删除实例
aws lightsail delete-instance --instance-name "${instanceName}"

# 创建实例
aws lightsail create-instances \
 --instance-names "${instanceName}" \
 --availability-zone ap-northeast-1a \
 --blueprint-id "${blueprintId}" \
 --bundle-id "${bundleId}" \
 --key-pair-name "${keyPairName}"

# 等待实例创建并获取IP地址
ipAddress=""
while [ -z "$ipAddress" ]; do
  echo "Waiting for instance to have a public IP address..."
  sleep 1  # 等待1秒
  ipAddress=$(aws lightsail get-instance --instance-name "${instanceName}" | jq -r .instance.publicIpAddress)
done

# 等待实例的SSH端口开放
echo "Instance IP address: ${ipAddress}"
while ! nc -zv "${ipAddress}" 22 2>&1 | grep -q succeeded; do
  echo "Waiting for SSH port to be available..."
  sleep 1  # 等待1秒
done

# 连接实例
echo "Connecting to instance..."
ssh -i ~/.ssh/"${keyPairName}".pem ubuntu@"${ipAddress}"
```

## 安装 outline

```bash
sudo bash -c "$(wget -qO- https://raw.githubusercontent.com/Jigsaw-Code/outline-server/master/src/server_manager/install_scripts/install_server.sh)"
```

## 根据 outline 提示开放端口

```bash
tcpPort=""
udpPort=""

aws lightsail open-instance-public-ports \
 --instance-name ubuntu \
 --port-info fromPort="${tcpPort}",toPort="${tcpPort}",protocol=TCP \
&& \
aws lightsail open-instance-public-ports \
 --instance-name ubuntu \
 --port-info fromPort="${udpPort}",toPort="${udpPort}",protocol=TCP \
&& \
aws lightsail open-instance-public-ports \
 --instance-name ubuntu \
 --port-info fromPort="${udpPort}",toPort="${udpPort}",protocol=UDP
```
