---
date: "2020-12-23T00:00:00.000Z"
description: Project X 的文档.
title: VMess
weight: 6
---

[VMess](../../../develop/protocols/vmess) 是一个加密传输协议，通常作为 Xray 客户端和服务器之间的桥梁。

{{% notice danger important %}}
VMess 依赖于系统时间，请确保使用 Xray 的系统 UTC 时间误差在 90 秒之内，时区无关。在 Linux 系统中可以安装`ntp`服务来自动同步系统时间。</br>
{{% /notice %}}

## InboundConfigurationObject
---

```json
{
  "clients": [
    {
      "id": "27848739-7e62-4138-9fd3-098a63964b6b",
      "level": 0,
      "alterId": 0,
      "email": "love@xray.com"
    }
  ],
  "default": {
    "level": 0,
    "alterId": 0
  },
  "detour": {
    "to": "tag_to_detour"
  },
  "disableInsecureEncryption": false
}
```

{{% notice dark %}} `clients`: \[ [ClientObject](#clientobject) \]{{% /notice %}}

一个数组，代表一组服务端认可的用户.

其中每一项是一个用户[ClientObject](#clientobject)。

当此配置用作动态端口时，Xray 会自动创建用户。

{{% notice dark %}} `detour`: [DetourObject](#detourobject){{% /notice %}}

指示对应的出站协议使用另一个服务器。

{{% notice dark %}} `default`: [DefaultObject](#defaultobject){{% /notice %}}

可选，clients 的默认配置。仅在配合`detour`时有效。

{{% notice dark %}} `disableInsecureEncryption`: true | false{{% /notice %}}

是否禁止客户端使用不安全的加密方式，如果设置为 true 当客户端指定下列加密方式时，服务器会主动断开连接。
- `"none"`
- `"aes-128-cfb"`

默认值为`false`。

<br />
### ClientObject
---

```json
{
  "id": "27848739-7e62-4138-9fd3-098a63964b6b",
  "level": 0,
  "alterId": 4,
  "email": "love@xray.com"
}
```

{{% notice dark %}} `id`: string{{% /notice %}}

VMess 的用户 ID。必须是一个合法的 UUID。

{{% notice dark %}} `level`: number{{% /notice %}}

用户等级，连接会使用这个用户等级对应的[本地策略](../../policy#levelpolicyobject)。

level 的值, 对应 [policy](../../policy#policyobject) 中 level 的值. 如不指定, 默认为 0.

{{% notice dark %}} `alterId`: number{{% /notice %}}

为了进一步防止被探测，一个用户可以在主 ID 的基础上，再额外生成多个 ID。这里只需要指定额外的 ID 的数量，推荐值为 0 代表启用 VMessAEAD。<br />
最大值 65535。这个值不能超过服务器端所指定的值。

不指定的话，默认值是 0。

{{% notice info %}}
**TIP**\
客户端 AlterID 设置为 0 代表启用 VMessAEAD ；服务端为自动适配，可同时兼容启用和未开启 VMessAEAD 的客户端。<br />
客户端可通过设置环境变量 Xray_VMESS_AEAD_DISABLED=true 强行禁用 VMessAEAD
{{% /notice %}}

{{% notice dark %}} `email`: string{{% /notice %}}

用户邮箱地址，用于区分不同用户的流量。

<br />

### DetourObject
---

```json
{
  "to": "tag_to_detour"
}
```

{{% notice dark %}} `to`: string{{% /notice %}}

一个 inbound 的`tag`, 指定的 inbound 的必须是使用 VMess 协议的 inbound. 

<br />

### DefaultObject
---

```json
{
  "level": 0,
  "alterId": 0
}
```

{{% notice dark %}} `level`: number{{% /notice %}}

用户等级，连接会使用这个用户等级对应的[本地策略](../../policy#levelpolicyobject)。

level 的值, 对应 [policy](../../policy#policyobject) 中 level 的值. 如不指定, 默认为 0.

{{% notice dark %}} `alterId`: number{{% /notice %}}

动态端口的默认`alterId`，默认值为`0`。

<br />

## VMess MD5 认证信息 玷污机制
---

为了进一步对抗可能的探测和封锁，每个 VMess 认证数据的服务端结构都会包含一个一次写入的玷污状态标记，初始状态为无瑕状态，当服务器检测到重放探测时或者因为其他原因入站连接出错以致校验数据不正确时，该连接所对应的请求认证数据会被玷污。

被玷污的认证数据无法被用于建立连接，当攻击者或客户端使用被玷污的认证数据建立连接时，服务器会输出包含 "invalid user" "ErrTainted" 的错误信息，并阻止该连接。

当服务器没有受到重放攻击时，该机制对正常连接的客户端没有影响。

如果服务器正在被重放攻击，可能会出现连接不稳定的情况。

{{% notice info %}}
**TIP**\
拥有服务器 UUID 以及其他连接数据的恶意程序可能根据此机制对服务器发起拒绝服务攻击，受到此类攻击的服务可以通过修改 proxy/vmess/validator.go 文件中 func (v \*TimedUserValidator) BurnTaintFuse(userHash []byte) error 函数的 atomic.CompareAndSwapUint32(pair.taintedFuse, 0, 1) 语句为 atomic.CompareAndSwapUint32(pair.taintedFuse, 0, 0) 来解除服务器对此类攻击的安全保护机制。使用 VMessAEAD 认证机制的客户端不受到 VMess MD5 认证信息 玷污机制 的影响。
{{% /notice %}}

