---
date: "2020-12-23T00:00:00.000Z"
description: Project X 的文档.
title: 路由
weight: 4
---

路由功能模块可以将入站数据按不同规则由不同的出站连接发出，以达到按需代理的目的。

如常见用法是分流国内外流量，Xray 可以通过内部机制判断不同地区的流量，然后将它们发送到不同的出站代理。

## RoutingObject
---
`RoutingObject` 对应配置文件的 `routing` 项。

```json
{
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [],
    "balancers": []
  }
}
```
{{% notice dark %}}`domainStrategy`: "AsIs" | "IPIfNonMatch" | "IPOnDemand"{{% /notice %}} 

域名解析策略，根据不同的设置使用不同的策略。

* `"AsIs"`：只使用域名进行路由选择。默认值。
* `"IPIfNonMatch"`：当域名没有匹配任何规则时，将域名解析成 IP（A 记录或 AAAA 记录）再次进行匹配；
  * 当一个域名有多个 A 记录时，会尝试匹配所有的 A 记录，直到其中一个与某个规则匹配为止；
  * 解析后的 IP 仅在路由选择时起作用，转发的数据包中依然使用原始域名；
* `"IPOnDemand"`：当匹配时碰到任何基于 IP 的规则，将域名立即解析为 IP 进行匹配；

{{% notice dark %}}`rules`: \[[RuleObject](#ruleobject)\]{{% /notice %}}

对应一个数组，数组中每个元素是一个规则。

对于每一个连接，路由将根据这些规则依次进行判断，当一个规则生效时，即将这个连接转发至它所指定的 `outboundTag`或 `balancerTag`。

{{% notice info %}}
**TIP**\
当没有匹配到任何规则时，流量默认由第一个outbound发出。
{{% /notice %}}

{{% notice dark %}}`balancers`: \[ [BalancerObject](#balancerobject) \]{{% /notice %}}
 
一个数组，数组中每个元素是一个负载均衡器的配置。

当一个规则指向一个负载均衡器时，Xray 会通过此负载均衡器选出一个outbound, 然后由它转发流量。

<br />
### RuleObject
---
```json
{
    "type": "field",
    "domain": [
        "baidu.com",
        "qq.com",
        "geosite:cn"
    ],
    "ip": [
        "0.0.0.0/8",
        "10.0.0.0/8",
        "fc00::/7",
        "fe80::/10",
        "geoip:cn"
    ],
    "port": "53,443,1000-2000",
    "sourcePort": "53,443,1000-2000",
    "network": "tcp",
    "source": [
        "10.0.0.1"
    ],
    "user": [
        "love@xray.com"
    ],
    "inboundTag": [
        "tag-vmess"
    ],
    "protocol": [
        "http",
        "tls",
        "bittorrent"
    ],
    "attrs": "attrs[':method'] == 'GET'",
    "outboundTag": "direct",
    "balancerTag": "balancer"
}
```

<br />
{{% notice danger important %}}
当多个属性同时指定时，这些属性需要 **同时** 满足，才可以使当前规则生效。
{{% /notice %}}
<br />  

{{% notice dark %}}`type`: "field"{{% /notice %}}

目前只支持`"field"`这一个选项。

{{% notice dark %}}`domain`: \[string\]{{% /notice %}}

一个数组，数组每一项是一个域名的匹配。有以下几种形式：

* 纯字符串：当此字符串匹配目标域名中任意部分，该规则生效。比如 "sina.com" 可以匹配 "sina.com"、"sina.com.cn" 和"www.sina.com"，但不匹配 "sina.cn"。
* 正则表达式：由 `"regexp:"` 开始，余下部分是一个正则表达式。当此正则表达式匹配目标域名时，该规则生效。例如 "regexp:\\\\.goo.*\\\\.com$" 匹配"www.google.com"或 "fonts.googleapis.com"，但不匹配 "google.com"。
* 子域名（推荐）：由 `"domain:"` 开始，余下部分是一个域名。当此域名是目标域名或其子域名时，该规则生效。例如 "domain:xray.com" 匹配"www.xray.com"、"xray.com"，但不匹配 "wxray.com"。
* 完整匹配：由 `"full:"` 开始，余下部分是一个域名。当此域名完整匹配目标域名时，该规则生效。例如 "full:xray.com" 匹配 "xray.com" 但不匹配"www.xray.com"。
* 预定义域名列表：由 `"geosite:"` 开头，余下部分是一个名称，如 `geosite:google` 或者 `geosite:cn`。名称及域名列表参考 [预定义域名列表](#预定义域名列表)。
* 从文件中加载域名：形如 `"ext:file:tag"`，必须以 `ext:`（小写）开头，后面跟文件名和标签，文件存放在 [资源目录](../env#资源文件路径) 中，文件格式与 `geosite.dat` 相同，标签必须在文件中存在。
 {{% notice info %}}
**TIP**\
`"ext:geoip.dat:cn"` 等价于 `"geoip:cn"`
{{% /notice %}}

{{% notice dark %}}`ip`: \[string\]{{% /notice %}}

一个数组，数组内每一个元素代表一个 IP 范围。当某一元素匹配目标 IP 时，此规则生效。有以下几种形式：

* IP：形如 `"127.0.0.1"`。
* [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing)：形如 `"10.0.0.0/8"`。
* 预定义IP列表：此列表预置于每一个 Xray 的安装包中，文件名为 `geoip.dat`。使用方式形如 `"geoip:cn"`，必须以 `geoip:`（小写）开头，后面跟双字符国家代码，支持几乎所有可以上网的国家。
  * 特殊值：`"geoip:private"`，包含所有私有地址，如 `127.0.0.1`。
* 从文件中加载 IP：形如 `"ext:file:tag"`，必须以 `ext:`（小写）开头，后面跟文件名和标签，文件存放在 [资源目录](env.md#资源文件路径) 中，文件格式与 `geoip.dat` 相同标签必须在文件中存在。

{{% notice dark %}}`port`：number | string{{% /notice %}}

目标端口范围，有三种形式：

* `"a-b"`：a 和 b 均为正整数，且小于 65536。这个范围是一个前后闭合区间，当目标端口落在此范围内时，此规则生效。
* `a`：a 为正整数，且小于 65536。当目标端口为 a 时，此规则生效。
* 以上两种形式的混合，以逗号 "," 分隔。形如：`"53,443,1000-2000"`。

{{% notice dark %}}`sourcePort`：number | string{{% /notice %}}

来源端口，有三种形式：

* `"a-b"`：a 和 b 均为正整数，且小于 65536。这个范围是一个前后闭合区间，当目标端口落在此范围内时，此规则生效。
* `a`：a 为正整数，且小于 65536。当目标端口为 a 时，此规则生效。
* 以上两种形式的混合，以逗号 "," 分隔。形如：`"53,443,1000-2000"`。

{{% notice dark %}}`network`: "tcp" | "udp" | "tcp,udp"{{% /notice %}}

可选的值有 "tcp"、"udp" 或 "tcp,udp"，当连接方式是指定的方式时，此规则生效。

{{% notice dark %}}`source`: \[string\]{{% /notice %}}

一个数组，数组内每一个元素是一个 IP 或 CIDR。当某一元素匹配来源 IP 时，此规则生效。

{{% notice dark %}}`user`: \[string\]{{% /notice %}}

一个数组，数组内每一个元素是一个邮箱地址。当某一元素匹配来源用户时，此规则生效。

{{% notice dark %}}`inboundTag`: \[string\]{{% /notice %}}

一个数组，数组内每一个元素是一个标识。当某一元素匹配入站协议的标识时，此规则生效。

{{% notice dark %}}`protocol`: \[ "http" | "tls" | "bittorrent" \]{{% /notice %}}

一个数组，数组内每一个元素表示一种协议。当某一个协议匹配当前连接的协议类型时，此规则生效。
 {{% notice info %}}
**TIP**\
必须开启入站代理中的 `sniffing` 选项, 才能嗅探出连接所使用的协议类型. 
{{% /notice %}}

{{% notice dark %}}`attrs`: string{{% /notice %}}

一段脚本，用于检测流量的属性值。当此脚本返回真值时，此规则生效。

脚本语言为 [Starlark](https://github.com/bazelbuild/starlark)，它的语法是 Python 的子集。脚本接受一个全局变量 `attrs`，其中包含了流量相关的属性。

目前只有 http 入站代理会设置这一属性。

示例：

* 检测 HTTP GET：`"attrs[':method'] == 'GET'"`
* 检测 HTTP Path：`"attrs[':path'].startswith('/test')"`
* 检测 Content Type：`"attrs['accept'].index('text/html') >= 0"`

{{% notice dark %}}`outboundTag`: string{{% /notice %}}

对应一个outbound的标识。

{{% notice dark %}}`balancerTag`: string{{% /notice %}}

对应一个Balancer的标识。

 {{% notice info %}}
**TIP**\
`balancerTag` 和 `outboundTag` 须二选一。当同时指定时，`outboundTag` 生效。
{{% /notice %}}


<br />
### BalancerObject
---
负载均衡器配置。当一个负载均衡器生效时，它会从指定的outbound中，按配置选出一个最合适的outbound，进行流量转发。

```json
{
    "tag": "balancer",
    "selector": []
}
```

{{% notice dark %}}`tag`: string{{% /notice %}}

此负载均衡器的标识，用于匹配 `RuleObject` 中的 `balancerTag`。

{{% notice dark %}}`selector`: \[ string \]{{% /notice %}}


一个字符串数组，其中每一个字符串将用于和outbound标识的前缀匹配。在以下几个outbound标识中：`[ "a", "ab", "c", "ba" ]`，`"selector": ["a"]` 将匹配到 `[ "a", "ab" ]`。

如果匹配到多个outbound，负载均衡器目前会从中随机选出一个作为最终的outbound。

<br />
### 预定义域名列表
---
此列表预置于每一个 Xray 的安装包中，文件名为 `geosite.dat`。这个文件包含了一些常见的域名，使用方式：`geosite:filename`，如 `geosite:google` 表示对文件内符合 `google` 内包含的域名，进行路由筛选或 DNS 筛选。

常见的域名有：

* `category-ads`：包含了常见的广告域名。
* `category-ads-all`：包含了常见的广告域名，以及广告提供商的域名。
* `cn`：相当于 `geolocation-cn` 和 `tld-cn` 的合集。
* `apple`：包含了 Apple 旗下绝大部分域名。
* `google`：包含了 Google 旗下绝大部分域名。
* `microsoft`：包含了 Microsoft 旗下绝大部分域名。
* `facebook`：包含了 Facebook 旗下绝大部分域名。
* `twitter`：包含了 Twitter 旗下绝大部分域名。
* `telegram`：包含了 Telegram 旗下绝大部分域名。
* `geolocation-cn`：包含了常见的大陆站点域名。
* `geolocation-!cn`：包含了常见的非大陆站点域名，同时包含了 `tld-!cn`。
* `tld-cn`：包含了 CNNIC 管理的用于中国大陆的顶级域名，如以 `.cn`、`.中国` 结尾的域名。
* `tld-!cn`：包含了非中国大陆使用的顶级域名，如以 `.hk`（香港）、`.tw`（台湾）、`.jp`（日本）、`.sg`（新加坡）、`.us`（美国）`.ca`（加拿大）等结尾的域名。
