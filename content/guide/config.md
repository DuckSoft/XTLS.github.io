---
date: "2020-12-23T00:00:00.000Z"
description: Project X 的文档.
title: 配置运行
weight: 2
---

[下载并安装](../install) 了 Xray 之后，您需要对它进行一下配置。

为了演示，这里只介绍简单的配置方式.

如需配置更复杂的功能，请参考更详细的 [配置文件](../../config) 中相关说明。

<br />

## 服务端配置

---

你需要一台防火墙外的服务器，来运行服务器端的 Xray。配置如下：

```json
{
  "inbounds": [
    {
      "port": 10086, // 服务器监听端口
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "b831381d-6324-4d53-ad4f-8cda48b30811"
          }
        ]
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

服务器的配置中需要确保 `id` 和端口与客户端一致，就可以正常连接了。

<br />

## 客户端配置

---

在你的 PC（或手机）中，需要用以下配置运行 Xray ：

```json
{
  "inbounds": [
    {
      "port": 1080, // SOCKS 代理端口，在浏览器中需配置代理并指向这个端口
      "listen": "127.0.0.1",
      "protocol": "socks",
      "settings": {
        "udp": true
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "server", // 服务器地址，请修改为你自己的服务器 ip 或域名
            "port": 10086, // 服务器端口
            "users": [
              {
                "id": "b831381d-6324-4d53-ad4f-8cda48b30811"
              }
            ]
          }
        ]
      }
    },
    {
      "protocol": "freedom",
      "tag": "direct"
    }
  ],
  "routing": {
    "domainStrategy": "IPOnDemand",
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:private"],
        "outboundTag": "direct"
      }
    ]
  }
}
```

上述配置唯一要更改的地方是你的服务器 IP，配置中已注明。上述配置会把除局域网（比如访问路由器）以外的所有流量转发至你的服务器。

<br />

## 运行

---

- 在 Windows 和 macOS 中，配置文件通常是 Xray 同目录下的 `config.json` 文件。
    - 直接运行 `Xray` 或 `Xray.exe` 即可。
- 在 Linux 中，配置文件通常位于 `/etc/Xray/` 或 `/usr/local/etc/Xray/` 目录下。
    - 运行 `xray run -c /etc/Xray/config.json`
    - 或使用 systemd 等工具将 Xray 作为服务在后台运行。

更多详细的说明可以参考 [配置文档](../../config) 和 [使用心得](../../documents)。
