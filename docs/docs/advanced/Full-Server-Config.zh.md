# 完整服务端配置

本页面提供了关于服务器配置文件中每一个字段的详细说明。

> **注意：** 在客户端和服务端配置中的多处都有 "类型选择" 的配置格式：

```yaml
example:
  type: a
  a:
    something: something
  b:
    something: something
  c:
    something: something
```

`type` 用于确定使用哪种模式以及解析哪些子字段。在这个例子中，`example` 字段可以是 `a`、`b` 或 `c`。如果选择了 `a`，则会解析 `a` 子字段，而 `b` 和 `c` 子字段则会被忽略。

## 监听地址

`listen` 是服务器的监听地址。如果省略，服务器将默认监听 `:443` 端口，这是 HTTP/3 的默认端口。

```yaml
listen: :443 # (1)!
```

1. 当只有端口没有 IP 地址时，服务器将监听所有可用的 IP 地址。

## TLS

可以选择使用 `tls` 或 `acme`，但不能同时包含两者。

=== "TLS"

    ```yaml
    tls:
      cert: some.crt
      key: some.key
    ```

=== "ACME"

    ```yaml
    acme:
      domains:
        - domain1.com
        - domain2.org
      email: your@email.net
      ca: zerossl # (1)!
      disableHTTP: false # (2)!
      disableTLSALPN: false # (3)!
      altHTTPPort: 8080 # (4)!
      altTLSALPNPort: 4443 # (5)!
      dir: my_acme_dir # (6)!

    ```

    1. 要使用的 CA。可以是 `letsencrypt` 或 `zerossl`。
    2. 禁用 HTTP 挑战。
    3. 禁用 TLS-ALPN 挑战。
    4. 备用的 HTTP 挑战端口。
    5. 备用的 TLS-ALPN 挑战端口。
    6. 存储 ACME 账户密钥和证书的目录。

## 混淆

默认 Hysteria 协议伪装为 HTTP/3。如果你的网络针对性屏蔽了 QUIC 或 HTTP/3 流量（但允许其他 UDP 流量），可以使用混淆来解决此问题。目前有一个名为 "Salamander" 的混淆实现，将数据包混淆成没有特征的 UDP 包。此功能需要一个混淆密码，密码在客户端和服务端必须相同。

> **注意:** 启用混淆将使服务器与标准的 QUIC 连接不兼容，失去 HTTP/3 伪装的能力。

```yaml
obfs:
  type: salamander
  salamander:
    password: cry_me_a_r1ver # (1)!
```

1. 替换为你的混淆密码。

## QUIC 参数

```yaml
quic:
  initStreamReceiveWindow: 8388608 # (1)!
  maxStreamReceiveWindow: 8388608 # (2)!
  initConnReceiveWindow: 20971520 # (3)!
  maxConnReceiveWindow: 20971520 # (4)!
  maxIdleTimeout: 30s # (5)!
  maxIncomingStreams: 1024 # (6)!
  disablePathMTUDiscovery: false # (7)!
```

1. 初始的 QUIC 流接收窗口大小。
2. 最大的 QUIC 流接收窗口大小。
3. 初始的 QUIC 连接接收窗口大小。
4. 最大的 QUIC 连接接收窗口大小。
5. 最长空闲超时时间。服务器会在多长时间没有收到任何客户端数据后关闭连接。
6. 最大并发传入流的数量。
7. 禁用 MTU 探测。

默认的流和连接接收窗口大小分别为 8MB 和 20MB。**除非你完全明白自己在做什么，否则不建议修改这些值。**如果要改，建议保持流接收窗口与连接接收窗口的比例为 2:5。

## 带宽

```yaml
bandwidth:
  up: 1 gbps
  down: 1 gbps
```

服务器端带宽值的作用是最高速度限制，限制服务器对每个客户端发送和接收的最大速率。**注意，服务器上传速度对应的是客户端下载速度，反之亦然。**可以选择不写这个字段，或者在一边或两边设为零，零代表没有限制。

支持的单位有：

- `bps` 或 `b`（每秒比特数）
- `kbps` 或 `kb` 或 `k`（每秒千比特）
- `mbps` 或 `mb` 或 `m`（每秒兆比特）
- `gbps` 或 `gb` 或 `g`（每秒吉比特）
- `tbps` 或 `tb` 或 `t`（每秒太比特）

### 忽略客户端带宽设置

```yaml
ignoreClientBandwidth: false
```

`ignoreClientBandwidth` 启用后，服务器将忽略客户端设置的任何带宽，永远使用传统的拥塞控制算法（目前为 BBR）。

这个功能主要为不希望让用户自己设置带宽的服务器提供。

### 带宽行为详解

**(本节中的信息是 Hysteria 的内部实现细节，可能会在不同版本之间发生变化)**

目前，Hysteria 有两种拥塞控制算法：

**BBR：** 由 Google 为 TCP 开发，我们对其进行了修改以移植到 QUIC。BBR 是标准的拥塞控制算法，包括慢启动和基于 RTT 变化的带宽估算。BBR 能独立运行，不需要用户手动设置带宽。

**Brutal：** 这是 Hysteria 自有的拥塞控制算法。与 BBR 不同，Brutal 采用固定速率模型，丢包或 RTT 变化不会降低速度。相反，如果无法达到预定的目标速率，反而会根据计算的丢包率提高发送速率来进行补偿。Brutal 只在你知道（并正确设置了）当前网络的理论最大速度时才能正常运行。其擅长在拥塞的网络中抢占带宽，因此得名。

> Brutal 如果带宽设置低于实际最大值也能正常运行；相当于限速。重要的是不要将其设置得高于实际最大值，否则会因为补偿机制导致连接速度更慢且不稳定。

拥塞控制控制的是连接的发送部分。从客户端的视角，如果没有设置 down 带宽值（但提供了 up），Hysteria 服务器会使用 BBR 向客户端发送数据，但客户端会使用 Brutal 向服务器上传数据，反之亦然。客户端可以提供两者，这样两个方向都会使用 Brutal，或者都不提供，这样两个方向都会使用 BBR。

如上所述，一个特殊情况是当服务器启用了 `ignoreClientBandwidth` 选项，无论客户端的带宽值如何，双方始终都会使用 BBR。

**目前，服务端的带宽限制限制的只是 Brutal 最大速率。因此在使用 BBR 的情况下并不会生效。**

## UDP

```yaml
disableUDP: false
```

`disableUDP` 启用后服务端禁用 UDP 转发，只支持 TCP。

```yaml
udpIdleTimeout: 60s
```

`udpIdleTimeout` 用于指定服务器对于每个 UDP 会话，在没有流量时保持本地 UDP 端口的时间长度。概念上与 NAT 的 UDP 会话超时时间相似。

## 验证

```yaml
auth:
  type: password
  password: your_password # (1)!
  userpass: # (2)!
    user1: pass1
    user2: pass2
    user3: pass3
  http:
    url: http://your.backend.com/auth # (3)!
    insecure: false # (4)!
  command: /etc/some_command # (5)!
```

1. 用自己选的强密码进行替换。
2. 用户名-密码映射表。
3. 处理验证的后端 URL。
4. 禁用后端服务器的 TLS 验证（仅适用于 HTTPS）。
5. 处理验证的命令路径。

### HTTP 验证

当使用 HTTP 验证，客户端尝试连接时，服务器会向后端服务器发送一个带有以下 JSON 的 `POST` 请求：

```json
{
  "addr": "123.123.123.123:44556", // (1)!
  "auth": "something_something", // (2)!
  "tx": 123456 // (3)!
}
```

1. 客户端的地址和端口。
2. 客户端提交的密码。
3. 客户端请求的发送速率。

后端服务器必须返回以下 JSON：

```json
{
  "ok": true, // (1)!
  "id": "john_doe" // (2)!
}
```

1. 是否允许客户端连接。
2. 客户 ID。用于日志和流量统计 API。

> **注意：** HTTP 状态码必须为 200，才认为验证成功。其他状态码会被认为是验证失败。

### 命令验证

当使用命令验证，客户端尝试连接时，服务器将执行指定的命令并附带以下参数：

```bash
/etc/some_command addr auth tx # (1)!
```

1. 每个参数的定义与上文中的 HTTP 验证部分相同。

该命令必须将客户 ID 输出到 `stdout`，如果允许客户端连接，则返回退出代码 0；如果客户端被拒绝，则返回非零。

如果命令执行失败，客户端将被拒绝。

## DNS 解析

可以指定用于解析客户端请求中的域名的 DNS 服务器。

```yaml
resolver:
  type: udp
  tcp:
    addr: 8.8.8.8:53 # (1)!
    timeout: 4s # (2)!
  udp:
    addr: 8.8.4.4:53 # (3)!
    timeout: 4s
  tls:
    addr: 1.1.1.1:853 # (4)!
    timeout: 10s
    sni: cloudflare-dns.com # (5)!
    insecure: false # (6)!
  https:
    addr: 1.1.1.1:443 # (7)!
    timeout: 10s
    sni: cloudflare-dns.com
    insecure: false
```

1. TCP DNS 服务器地址。
2. DNS 查询超时时间。
3. UDP DNS 服务器地址。
4. DNS over TLS 服务器地址。
5. DNS over TLS 服务器的 SNI。
6. 禁用 TLS 证书验证。
7. DNS over HTTPS 服务器地址。

如果省略，Hysteria 将使用系统默认的 DNS 服务器。

## ACL

ACL 是 Hysteria 服务端中一个非常强大的功能，可以用来自定义处理客户端请求的方式，往往配合出站规则（outbounds）使用。例如，可以使用 ACL 来屏蔽某些地址，或者针对不同网站使用不同的出口。

关于语法、使用方法以及其他信息，请参考 [ACL 文档](ACL.md)。

可以选择使用 `file` 或者 `inline`，但不能同时使用两者。

=== "文件"

    ```yaml
    acl:
      file: some.txt # (1)!
      # geoip: GeoLite2-Country.mmdb (2)
    ```

    1. ACL 文件的路径。
    2. 可选。取消注释以启用。GeoIP 数据库文件的路径。**如果省略这个字段，Hysteria 会自动下载最新的数据库。**

=== "内联"

    ```yaml
    acl:
      inline: # (1)!
        - reject(v2ex.com)
        - reject(*.v2ex.com)
        - reject(all, udp/443)
        - reject(geoip:cn)
      # geoip: GeoLite2-Country.mmdb (2)
    ```

    1. 内联 ACL 规则的列表。
    2. 可选。取消注释以启用。GeoIP 数据库文件的路径。**如果省略这个字段，Hysteria 会自动下载最新的数据库。**

> **注意：** Hysteria 只支持 MaxMind 的 GeoLite2 Country MMDB 格式的数据库来实现 GeoIP 功能。如果你不知道如何获取正确的文件，可以省略 geoip 字段，让 Hysteria 自动下载最新版本。只有在 ACL 中至少有一个 GeoIP 规则时，才会下载数据库。

## 出站规则 (Outbounds)

出站规则用于定义连接应通过哪个“出口”。例如，[与 ACL 结合使用时](ACL.md)，可以实现除 Netflix 外的所有流量都直接使用服务器本地网络，而 Netflix 流量通过另一个 SOCKS5 代理连接。

目前，Hysteria 支持以下几种出站类型：

- `direct`：通过本地网络直接连接。
- `socks5`：SOCKS5 代理。

**如果不使用 ACL，所有连接将始终通过列表中的第一个（“默认”）出站规则进行路由，所有其他出站规则将被忽略。**

```yaml
outbounds:
  - name: my_outbound_1 # (1)!
    type: direct
  - name: my_outbound_2
    type: socks5
    socks5:
      addr: shady.proxy.ru:1080 # (2)!
      username: hackerman # (3)!
      password: Elliot Alderson # (4)!
```

1. 出站规则的名称。在 ACL 中使用。
2. SOCKS5 代理地址。
3. 可选。SOCKS5 代理用户名。
4. 可选。SOCKS5 代理密码。

### 关于 `direct` 出站

`direct` 出站规则有一些额外的选项，可以用来定制其行为：

> **注意：** `bindIPv4`、`bindIPv6` 和 `bindDevice` 三者是互斥的。可以只指定 `bindIPv4` 和/或 `bindIPv6`，或者只使用 `bindDevice`。

```yaml
outbounds:
  - name: hoho
    type: direct
    direct:
      mode: auto # (1)!
      bindIPv4: 2.4.6.8 # (2)!
      bindIPv6: 0:0:0:0:0:ffff:0204:0608 # (3)!
      bindDevice: eth233 # (4)!
```

1. 详细解释见下文。
2. 要绑定的本地 IPv4 地址。
3. 要绑定的本地 IPv6 地址。
4. 要绑定的本地网卡。

支持的 `mode` 值包括：

- `auto`：默认值。双栈模式。客户端会同时尝试使用 IPv4 和 IPv6 地址连接目标，选择第一个成功的。
- `64`：优先使用 IPv6，如果没有可用的 IPv6 地址，使用 IPv4。
- `46`：优先使用 IPv4，如果没有可用的 IPv4 地址，使用 IPv6。
- `6`：始终使用 IPv6。如果没有可用的 IPv6 地址，连接失败。
- `4`：始终使用 IPv4。如果没有可用的 IPv4 地址，连接失败。

## 流量统计 API (HTTP)

流量统计 API 允许你通过 HTTP API 查询服务器的流量统计信息，以及踢用户下线。具体使用方法请参考 [流量统计 API 文档](Traffic-Stats-API.md)。

> **注意：** 此 API 没有验证，注意不要暴露在公网上。另外建议通过 ACL 阻止客户端访问。

```yaml
trafficStats:
  listen: :9999 # (1)!
```

1. 监听地址。

## 伪装 (Masquerade)

Hysteria 抵抗审查的关键之一就是它能伪装成标准的 HTTP/3 流量。这意味着数据包不仅对中间设备（middleboxes）看起来像是 HTTP/3，服务器还会像普通网站服务器一样真的响应 HTTP 请求。然而，这意味着你的服务器必须实际提供一些内容，以便看起来是真实的。

**如果审查不是你需要担心的问题，可以从配置文件中完全删除 `masquerade` 部分。这种情况下，Hysteria 将对所有 HTTP 请求始终返回 404 Not Found 。**

目前 Hysteria 提供以下几种伪装模式：

- `file`：作为一个静态文件服务器，从一个目录提供内容。
- `proxy`：作为一个反向代理，从另一个网站提供内容。

```yaml
masquerade:
  type: proxy
  file:
    dir: /www/masq # (1)!
  proxy:
    url: https://some.site.net # (2)!
    rewriteHost: true # (3)!
```

1. 用于提供文件的目录。
2. 要代理的网站的 URL。
3. 是否重写 `Host` 头以匹配被代理的网站。如果目标网站通过 `Host` 识别请求的网站，这个选项是必须的。

可以通过特定参数启动 Chrome 以强制使用 QUIC，测试你的伪装配置：

```bash
chrome --origin-to-force-quic-on=your.site.com:443 # (1)!
```

1. 用你服务器的域名替换。

> **注意：** 在用参数启动 Chrome 之前，请先确保完全退出了 Chrome，没有任何 Chrome 进程还在后台运行。否则参数可能不会生效。

然后访问 `https://your.site.com` 验证伪装是否生效。

### HTTP/HTTPS 伪装

通常支持 HTTP/3 的网站只是将其作为一个升级选项，在 80/443 端口上也提供 TCP 的 HTTP/HTTPS。如果希望模仿这种模式，可以使用 `listenHTTP` 和 `listenHTTPS` 选项来启用 HTTP/HTTPS 伪装。这种情况下，不需要用上述特殊参数启动 Chrome，和普通的网站一样访问即可验证伪装。

```yaml
masquerade:
  # ... (上述其他字段)
  listenHTTP: :80 # (1)!
  listenHTTPS: :443 # (2)!
  forceHTTPS: true # (3)!
```

1. HTTP (TCP) 监听地址。
2. HTTPS (TCP) 监听地址。
3. 是否强制使用 HTTPS。如果启用，HTTP 请求将被重定向到 HTTPS。

> **注意：** 目前没有迹象表明有任何政府/商业防火墙在利用 "缺少 TCP HTTP/HTTPS" 这点来检测 Hysteria 服务器。本功能仅为执着于 "做戏做全套" 的用户提供。既然要 "做戏做全套"，就没有理由将 HTTP/HTTPS 监听在 80/443 之外的自定义端口上（虽然 Hysteria 允许自定义监听地址）。
