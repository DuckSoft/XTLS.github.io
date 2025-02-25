---
date: "2020-12-23T00:00:00.000Z"
description: Project X 的文档.
title: Trojan
weight: 8
---

[Trojan](https://trojan-gfw.github.io/trojan/protocol) 协议

{{% notice danger important %}}
Trojan 被设计工作在正确配置的加密 TLS 隧道
{{% /notice %}}

## OutboundConfigurationObject

---

```json
{
  "servers": [
    {
      "address": "127.0.0.1",
      "port": 1234,
      "password": "password",
      "email": "love@xray.com",
      "flow": "xtls-rprx-direct",
      "level": 0
    }
  ]
}
```

{{% notice dark %}} `servers`: \[ [ServerObject](#serverobject) \]{{% /notice %}}

一个数组，其中每一项是一个 [ServerObject](#serverobject)。

<br />

### ServerObject

---

```json
{
  "address": "127.0.0.1",
  "port": 1234,
  "password": "password",
  "email": "love@xray.com",
  "flow": "xtls-rprx-direct",
  "level": 0
}
```

{{% notice dark %}} `address`: address{{% /notice %}}

服务端地址，支持 IPv4、IPv6 和域名。必填。

{{% notice dark %}} `port`: number{{% /notice %}}

服务端端口，通常与服务端监听的端口相同。

{{% notice dark %}} `password`: string{{% /notice %}}

密码. 必填，任意字符串。

{{% notice dark %}} `email`: string{{% /notice %}}

邮件地址，可选，用于标识用户

{{% notice dark %}} `flow`: string{{% /notice %}}

流控模式，用于选择 XTLS 的算法。

目前出站协议中有以下流控模式可选：

- `xtls-rprx-origin`：最初的流控模式。该模式纪念价值大于实际使用价值
- `xtls-rprx-origin-udp443`：同 `xtls-rprx-origin`, 但放行了目标为 443 端口的 UDP 流量
- `xtls-rprx-direct`：所有平台皆可使用的典型流控模式
- `xtls-rprx-direct-udp443`：同 `xtls-rprx-direct`, 但是放行了目标为 443 端口的 UDP 流量
- `xtls-rprx-splice`：Linux 平台下最建议使用的流控模式
- `xtls-rprx-splice-udp443`：同 `xtls-rprx-splice`, 但是放行了目标为 443 端口的 UDP 流量

{{% notice warning %}}
**注意**

当 `flow` 被指定时，还需要将该出站协议的 `streamSettings.security` 一项指定为 `xtls`，`tlsSettings` 改为 `xtlsSettings`。详情请参考 [streamSettings](../../transport#streamsettingsobject)。

此外，目前 XTLS 仅支持 TCP、mKCP、DomainSocket 这三种传输方式。
{{% /notice %}}

{{% notice %}}
**关于 `xtls-rprx-*-udp443` 流控模式**

VLESS 和 Trojan 协议使用 TCP 传输 UDP，即 UDP over TCP，但 XTLS 不会对 UoT 的数据进行处理。所以为了防止上层应用使用 QUIC，启用 XTLS 时客户端会自动拦截 UDP/443 的请求。若不需要拦截，请在客户端填写 `xtls-rprx-*-udp443`，服务端不变。
{{% /notice %}}

{{% notice danger important %}}
Splice 是 Linux Kernel 提供的函数，系统内核直接转发 TCP，不再经过 Xray 的内存，大大减少了数据拷贝、CPU 上下文切换的次数。

Splice 模式的的使用限制：

- Linux 环境
- 入站协议为 `Dokodemo door`、`Socks`、`HTTP` 等纯净的 TCP 连接, 或其它使用了 XTLS 的入站协议
- 出站协议为 VLESS + XTLS 或 Trojan + XTLS

此外，使用 Splice 时网速显示会滞后，这是特性，不是 bug。

需要注意的是，使用 mKCP 协议时不会使用 Splice（是的，虽然没有报错，但实际上根本没用到）。
{{% /notice %}}

{{% notice dark %}} `level`: number{{% /notice %}}

用户等级，连接会使用这个用户等级对应的[本地策略](../../policy#levelpolicyobject)。

level 的值, 对应 [policy](../../policy#policyobject) 中 level 的值. 如不指定, 默认为 0.
