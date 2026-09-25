---
title: linux安装docker及配置v2ray服务器
tags:
- docker
- linux
---

# 前言

主要参考docker的官方文档，`Docker Engine > install > Ubuntu`部分

https://docs.docker.com/engine/

# 安装

比较简单，按照官网步骤不会有问题

1. 配置docker的apt源

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

2. 通过apt安装

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

2. 运行hello world验证安装成功

```bash
sudo docker run hello-world
```

# 安装v2ray

安装参考https://github.com/v2fly/docker

指导文档https://www.v2ray.com/chapter_00/install.html

执行

```bash
# 拉取镜像
docker pull v2fly/v2fly-core
# 查看v2fly的命令帮助，--rm为容器运行结束后自动删除
docker run --rm v2fly/v2fly-core help
# 先测试配置文件格式是否正确
docker run --rm --name v2ray -v /home/ubuntu/v2fly_v4.config.json:/etc/v2ray/config.json -p 10810:10810 v2fly/v2fly-core test -c /etc/v2ray/config.json 
# 后台运行容器，将对应文件挂载到容器内，并且映射容器内部端口至外部
docker run -d --name v2ray2 -v /home/ubuntu/v2fly_v4.config.json:/etc/v2ray/config.json -p 10810:10810 v2fly/v2fly-core run -c /etc/v2ray/config.json
docker run -d --name v2ray -v ~/v2fly_v4.config.json:/etc/v2ray/config.json -p 10810:10810 v2fly/v2fly-core run -c /etc/v2ray/config.json
# 查看容器状态
docker container list
```

示例配置文件如下

```json
{
    "log": {
        "access": "/var/log/v2ray/access.log",
        "error": "/var/log/v2ray/error.log",
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "listen": "0.0.0.0",
            "port": 10810,
            "protocol": "shadowsocks",
            "settings": {
                "method": "aes-256-gcm",
                "password": "你的密码",
                "networks": "tcp,udp"
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom"
        }
    ]
}
```

之后可通过

```bash
docker exec -it <containerId> tail -f /var/log/v2ray/access.log -n <number>
```

命令查看日志内容

# linux部署openvpn服务

## 安装openvpn

执行如下命令

```bash
apt update
apt install openvpn
```

## 生成ssl证书

```bash
# 首先生成密钥，如下代码生成了一个4096长度的私钥并且将内容通过des3加密后输出到文件ca.key中，需要输入对称加密密码
openssl genrsa -des3 -out ca.key 4096
# 
```

