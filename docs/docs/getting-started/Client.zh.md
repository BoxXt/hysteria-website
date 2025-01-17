# 客户端

本教程将指导你完成 Hysteria 客户端的配置。与服务端类似，Hysteria 十分灵活，本教程展示的选项只是全部选项的一小部分。我们将重点介绍 HTTP 和 SOCKS5 代理模式。**如需进一步定制，请参考 [完整客户端配置](../advanced/Full-Client-Config.md)。**

这些步骤在 Linux 环境中执行，但在其他平台上类似。

## 前提条件

- 一个能连接的 Hysteria 服务器

## 创建配置文件

假设你已经将可执行文件下载到了一个目录中，名字是 `hysteria-linux-amd64-avx`。在同目录下创建一个 `config.yaml` 文件。

> **注意**： 部分配置项值可能和 YAML 语法冲突。 例如， 类似于 `[2001:db8::1]:443` 的 IPv6 地址+端口会导致配置文件解析失败。 只需将值放在 `""` 中写成 `"[2001:db8::1]:443"` 即可解决这类问题。

**请务必根据你的服务器设置和需求替换这些值。**

```yaml
server: your.domain.net:443 # (1)!

auth: Se7RAuFZ8Lzg # (2)!

bandwidth: # (3)!
  up: 20 mbps
  down: 100 mbps

socks5:
  listen: 127.0.0.1:1080 # (4)!

http:
  listen: 127.0.0.1:8080 # (5)!
```

1. 替换为你服务器的地址
2. 替换为你在服务器上设置的密码
3. 有关带宽的更多信息，请参见[下面](#_4)
4. 替换为你希望 SOCKS5 代理监听的地址
5. 替换为你希望 HTTP 代理监听的地址

### 带宽

**请将 `up` 和 `down` 的带宽值设置为你当前网络环境的理论最大速度。为了达到最佳性能，请务必设置准确。** 虽然强烈建议填写，但如果实在不知道上下行中一侧或者两侧的带宽，可以只设置一侧的带宽，移除另一侧，或者两侧都移除。这些带宽设置所控制的行为在 [完整客户端配置 - 带宽部分](../advanced/Full-Client-Config.md#_5) 中有描述。

### TLS

如果你的服务器使用的是自签名证书，可以在配置文件中指定要信任的 CA，或者使用 `insecure` 选项来完全禁用验证。如果你选择 `insecure` 设置，我们强烈建议配合 `pinSHA256` 选项来验证服务器证书的指纹。

=== "CA"

    ```yaml
    tls:
      ca: ca.crt # (1)!
    ```

    1. 替换为 CA 证书文件的路径

=== "禁用验证"

    ```yaml
    tls:
      insecure: true
    ```

    > **警告：** 单独使用 `insecure` 是不推荐的，因为这会让你的连接容易受到中间人攻击（MITM）。请在下一个标签页查看更好的替代方案。

=== "禁用验证 + pinSHA256"

    ```yaml
    tls:
      insecure: true
      pinSHA256: BA:88:45:17:A1... # (1)!
    ```

    1. 可以使用 openssl 获取你证书的指纹：`openssl x509 -noout -fingerprint -sha256 -in your_cert.crt`

## 运行客户端

通过以下命令启动客户端：

=== "默认文件名（config.yaml）"

    ```bash
    ./hysteria-linux-amd64-avx
    ```

=== "自定义文件名"

    ```bash
    ./hysteria-linux-amd64-avx -c whatever.yaml
    ```

> **提示：** 你也可以使用 `./hysteria-linux-amd64-avx client`，但由于客户端是默认模式，所以这部分可以省略。

如果你看到日志显示 "connected to server" 且没有错误，恭喜 🎉！你已成功部署了一个 Hysteria 客户端。

你还会看到一个日志信息 "use this URI to share your server" 包含一个 URI。这个 URI 可以用作客户端配置文件中的 `server` 值。由于它已经包含了密码和一些其他设置，因此不再需要单独指定它们。有关 URI 格式的更多信息，请参考 [URI 格式](../developers/URI-Scheme.md)。

本教程不会详细介绍如何使用 HTTP 或 SOCKS5 代理，因为网上已有大量的教程。对于完全不了解代理的人，我们推荐使用 [Proxy SwitchyOmega 浏览器插件](https://chrome.google.com/webstore/detail/proxy-switchyomega/padekgcemlokbadohgkifijomclgjgif) 作为一个起点。
