---
title: ubuntu上部署vless+reality
date: 2026/08/09
---

# ubuntu上部署vless+reality

reality据说可以有效规避审查

通过telnet测试端口这个协议似乎是把请求伪装成网页数据，感觉会有点浪费带宽

## 安装xray

```bash
bash <(curl -Ls https://raw.githubusercontent.com/XTLS/Xray-install/main/install-release.sh)
# 确认安装成功
xray version
```

## 生成Reality密钥

```bash
xray x25519
# 用于服务器配置
# PrivateKey: xxxxxxxxxxxxxxxxxxxxxxxxx
# 用于客户端连接
# Password:  yyyyyyyyyyyyyyyyyyyyyyyyy
# Hash32:    zzzzzzzzzzz
```

## 配置服务

配置位置位于`/usr/local/etc/xray/config.json`

配置示例如下

```json
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "自己生成UUID",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "dest": "www.cloudflare.com:443",
          "serverNames": ["www.cloudflare.com"],
          "privateKey": "你的PrivateKey",
          "shortIds": ["自己决定，可以从uuid截一部分"]
        }
      }
    }
  ],
  "outbounds": [
    { "protocol": "freedom" }
  ]
}
```

## 启动服务

```bash
# 测试配置是否有效
xray run -test -config /usr/local/etc/xray/config.json
# 启动服务
systemctl restart xray
systemctl enable xray
systemctl status xray
```

## 客户端配置

客户端使用的v2rayN

配置如下

+ 别名：自定义
+ 地址：服务器ip
+ 端口：服务端配置文件上配置的端口
+ 用户id: 自己生成的uuid
+ 流控：服务端配置文件中的flow，如示例文件中的是xtls-rprx-vision
+ 传输协议： tcp
+ 伪装类型：none
+ 传输层安全：reality
+ SNI: realitySettings中的dest配置
+ fingerprint：客户端指纹，默认选chrome即可
+ PublicKey：之前生成的password
+ ShortId：服务端配置文件的shortId

