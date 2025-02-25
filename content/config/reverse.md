---
date: "2020-12-23T00:00:00.000Z"
description: Project X 的文档.
title: 反向代理
weight: 10
---

反向代理可以把服务器端的流量向客户端转发，即逆向流量转发。

反向代理的大致工作原理如下:

* 假设在主机 A 中有一个网页服务器，这台主机没有公网 IP，无法在公网上直接访问。另有一台主机 B，它可以由公网访问。现在我们需要把 B 作为入口，把流量从 B 转发到 A。
* 在主机 A 中配置 Xray，称为`bridge`，在 B 中也配置 Xray，称为 `portal`。
* `bridge` 会向 `portal` 主动建立连接，此连接的目标地址可以自行设定。`portal` 会收到两种连接，一是由 `bridge` 发来的连接，二是公网用户发来的连接。`portal` 会自动将两类连接合并。于是 `bridge` 就可以收到公网流量了。
* `bridge` 在收到公网流量之后，会将其原封不动地发给主机 A 中的网页服务器。当然，这一步需要路由的协作。
* `bridge` 会根据流量的大小进行动态的负载均衡。

{{% notice info %}}
**TIP**\
反向代理默认已开启 [Mux](mux.md)，请不要在其用到的outbound上再次开启 Mux。
{{% /notice %}}

{{% notice warning %}}
反向代理功能尚处于测试阶段，可能会有一些问题。
{{% /notice %}}

<br />
## ReverseObject
---
`ReverseObject` 对应配置文件的 `reverse` 项。

```json
{
  "reverse": {
    "bridges": [
      {
        "tag": "bridge",
        "domain": "test.xray.com"
      }
    ],
    "portals": [
      {
        "tag": "portal",
        "domain": "test.xray.com"
      }
    ]
  }
}
```

{{% notice dark %}} `bridges`: \[[BridgeObject](#bridgeobject)\]{{% /notice %}}


数组，每一项表示一个 `bridge`。每个 `bridge` 的配置是一个 [BridgeObject](#bridgeobject)。

{{% notice dark %}} `portals`: \[[PortalObject](#portalobject)\]{{% /notice %}}


数组，每一项表示一个 `portal`。每个 `portal` 的配置是一个 [PortalObject](#bridgeobject)。

<br />
### BridgeObject
---
```json
{
    "tag": "bridge",
    "domain": "test.xray.com"
}
```

{{% notice dark %}} `tag`: string{{% /notice %}}


所有由 `bridge` 发出的连接，都会带有这个标识。可以在 [路由配置](../routing) 中使用 `inboundTag` 进行识别。

{{% notice dark %}} `domain`: string{{% /notice %}}


指定一个域名，`bridge` 向 `portal` 建立的连接，都会使用这个域名进行发送。

这个域名只作为 `bridge` 和 `portal` 的通信用途，不必真实存在。

<br />
### PortalObject
---
```json
{
    "tag": "portal",
    "domain": "test.xray.com"
}
```

{{% notice dark %}} `tag`: string{{% /notice %}}


`portal` 的标识。在  [路由配置](../routing) 中使用 `outboundTag` 将流量转发到这个 `portal`。

{{% notice dark %}} `domain`: string{{% /notice %}}


一个域名。当 `portal` 接收到流量时，如果流量的目标域名是此域名，则 `portal` 认为当前连接上 `bridge` 发来的通信连接。而其它流量则会被当成需要转发的流量。`portal` 所做的工作就是把这两类连接进行识别并拼接。

{{% notice info %}}
**TIP**\
一个 Xray 既可以作为 `bridge`，也可以作为 `portal`，也可以同时两者，以适用于不同的场景需要。{{% /notice %}}

<br />
## 完整配置样例
---

{{% notice info %}}
**TIP**\
在运行过程中，建议先启用 `bridge`，再启用 `portal`。
{{% /notice %}}

### bridge配置
---
`bridge` 通常需要两个outbound，一个用于连接 `portal`，另一个用于发送实际的流量。也就是说，你需要用路由区分两种流量。

反向代理配置:

```json
{
    "bridges": [
        {
            "tag": "bridge",
            "domain": "test.xray.com"
        }
    ]
}
```

outbound:

```json
{
    "tag": "out",
    "protocol": "freedom",
    "settings": {
        "redirect": "127.0.0.1:80" // 将所有流量转发到网页服务器
    }
},
{
    "protocol": "vmess",
    "settings": {
        "vnext": [
            {
                "address": "portal 的 IP 地址",
                "port": 1024,
                "users": [
                    {
                        "id": "27848739-7e62-4138-9fd3-098a63964b6b"
                    }
                ]
            }
        ]
    },
    "tag": "interconn"
}
```

路由配置:

```json
"routing": {
    "rules": [
        {
            "type": "field",
            "inboundTag": [
                "bridge"
            ],
            "domain": [
                "full:test.xray.com"
            ],
            "outboundTag": "interconn"
        },
        {
            "type": "field",
            "inboundTag": [
                "bridge"
            ],
            "outboundTag": "out"
        }
    ]
}
```

<br />
### portal配置
---
`portal` 通常需要两个inbound，一个用于接收 `bridge` 的连接，另一个用于接收实际的流量。同时你也需要用路由区分两种流量。

反向代理配置:

```json
{
    "portals": [
        {
            "tag": "portal",
            "domain": "test.xray.com" // 必须和 bridge 的配置一样
        }
    ]
}
```

inbound:

```json
{
    "tag": "external",
    "port": 80, // 开放 80 端口，用于接收外部的 HTTP 访问
    "protocol": "dokodemo-door",
    "settings": {
        "address": "127.0.0.1",
        "port": 80,
        "network": "tcp"
    }
},
{
    "port": 1024, // 用于接收 bridge 的连接
    "tag": "interconn",
    "protocol": "vmess",
    "settings": {
        "clients": [
            {
                "id": "27848739-7e62-4138-9fd3-098a63964b6b"
            }
        ]
    }
}
```

路由配置:

```json
"routing": {
    "rules": [
        {
            "type": "field",
            "inboundTag": [
                "external"
            ],
            "outboundTag": "portal"
        },
        {
            "type": "field",
            "inboundTag": [
                "interconn"
            ],
            "outboundTag": "portal"
        }
    ]
}
```

