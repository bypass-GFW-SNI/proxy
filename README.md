 # bypass-GFW-SNI-proxy

DNS 方式请参考 [bypass-GFW-SNI/main](https://github.com/bypass-GFW-SNI/main)。

此“半” POC 程序可以在不自行架设境外服务器的情况下，直接访问符合特定条件的被中国防火长城（“GFW”）屏蔽的站点。TL;DR，是通过类似域前置的方式突破了 GFW 的 SNI 封锁。

## 特定条件

存在可以访问的上游无污染 DNS ，使其解析被封锁域名，且返回的 IP 可以正常进行 TLS 握手。

#### 其中：

1. 访问境外 DNS 皆存在 DNS 污染。但目前 GFW 仅污染 53 端口，并没有屏蔽使用 *DNS over TLS* 或者 *DNS over HTTPS* 技术的 DNS 服务商及相应端口，例如 `1.1.1.1` 或 `8.8.8.8`。

2. 大型境外被封禁网站多为直接封禁 IP。但目前 GFW 仅有 IPv4 黑名单，还未封禁 IPv6 地址，而国内大型运营商，例如中国联通，已开始分配 IPv6 地址，并可以通过其连接互联网。

3. 某些被封禁域名使用了非独立 IP 的 CDN，例如 Cloudflare 免费版，因此强制需要 SNI，否则无法成功握手。或者因为配置了多证书，导致返回的默认证书并非欲访问域名。

### 样例

<dl>
  <dt>有 IPv4 接口</dt>
  <dd>Amazon、Reddit、Steam、Wikipedia、Yahoo、Twitch 等。</dd>
  <dt>有 IPv6 接口</dt>
  <dd>Google、Youtube、Facebook 等。</dd>
  <dt>依然无法访问</dt>
  <dd>Twitter（因 IPv4 地址黑名单，且没有配置 IPv6）等，未使用 HTTPS 技术的网站，以及符合 <b>其中.3</b> 条件的域名。</dd>
</dl>

# 实现

### 定义

<dl>
  <dt>代理地址</dt>
  <dd>用户自行配置的监听 Socks5 代理地址</dd>
  <dt>上游无污染 DNS</dt>
  <dd>解析存在于自定域名列表里的域名时会请求的上游 DNS。</dd>
  <dt>自签发 CA 证书</dt>
  <dd>为本地中间人攻击而自签的 CA 证书。</dd>
  <dt>原网站证书</dt>
  <dd>原网站的正常证书。</dd>
  <dt>本地 XX</dt>
  <dd>程序内所运行的相应 XX 服务。</dd>
</dl>

### 准备

1. 系统信任自签发 CA 证书。
2. 启动程序，程序会监听代理地址。
3. 浏览器或系统代理设置为代理地址。

## 流程

假设浏览器通过代请求 `example.com`。本地服务器将会通过上游无污染 DNS 解析 `example.com`，并尝试和解析出的 IP 逐一进行 TLS 握手，且握手信息中的 Server Name 将被替换为 IP 地址本身，这可以使远程网络服务器返回默认的 TLS 证书。若握手失败，则表明连接超时，或者 IP 地址已被 GFW 列为黑名单，或者出现 <b>其中.3</b> 的情况；若握手成功，则程序将会使用原本的域名，即 `example.com`，校验原网站证书的有效性。若校验失败，则表明要么原网站被中间人攻击，要么出现 <b>其中.3</b> 的情况。

若一切成功，则表明已成功访问被封锁网站。此时浏览器将会和本地服务器进行 TLS 握手，而本地服务器将会根据握手中的 SNI 使用自签发 CA 证书签发 SSL 证书。此时本地 TLS 握手成功，并且程序将开始转发数据。

# 运行

基于 Go Debug 信息以及目前程序完整度，将暂时不考虑分发已编译二进制。

运行此程序，你需要在电脑上安装 [Go 运行环境](https://golang.org/dl)，同时 `go get` 下列包：

<dl>
  <dt><a href="https://github.com/Sirupsen/logrus">github.com/Sirupsen/logrus</a></dt>
  <dd>程序所使用的日志包。</dd>
  <dt><a href="https://github.com/armon/go-socks5">github.com/armon/go-socks5</a></dt>
  <dd>Socks5 代理服务器包。</dd>
  <dt><a href="https://github.com/miekg/dns">github.com/miekg/dns</a></dt>
  <dd>DNS 请求包。</dd>
  <dt><a href="https://godoc.org/golang.org/x/net/publicsuffix">golang.org/x/net/publicsuffix</a></dt>
  <dd>域名匹配以及证书签发所需的 Public Suffix 列表。</dd>
</dl>

最后，`go run main.go` 便可启动程序。详细流程参考 **实现—准备**。

## 配置

### 常量

`const` 中有 8 个可配置参数，分别为：

<dl>
  <dt>caCert</dt>
  <dd>自签发 CA 证书路径。</dd>
  <dt>caKey</dt>
  <dd>自签发 CA 证书所对应的私钥路径。</dd>
  <dt>gfwDNS</dt>
  <dd>上游无污染 DNS 地址（需要为 IP:端口 格式）。</dd>
  <dt>certExpire</dt>
  <dd>证书签发过期时间。</dd>
  <dt>dialTimeout</dt>
  <dd>TCP 握手超时时间。</dd>
  <dt>cacheAddrTtl</dt>
  <dd>可用解析 IP 缓存时长（TTL）。</dd>
  <dt>proxyAddr</dt>
  <dd>代理地址。</dd>
  <dt>logLevel</dt>
  <dd>日志详细度，参见<a href="https://godoc.org/github.com/sirupsen/logrus#Level">日志包文档</a>。</dd>
</dl>

#### 其中：

`caCert` 和 `caKey` 需要你的证书及私钥格式为 PEM。同时，`caKey` 默认你的私钥算法为 RSA。如果你的私钥算法不是 RSA，请自行修改 `var` 中 `caPriKey` 的变量类型，和 `init()` 函数中的相关调用。

与 DNS 有关的参数 `gfwDNS` 在更改时可能需要与 `var` 中的 `gfwDnsCli` 中的 `New` 函数所对应地同时进行更改。更详细地说，需要更改其中新建 `dns.Client` 的 `Net` 参数，其与 DNS 所须的请求方式有关。参见 [DNS 包文档](https://godoc.org/github.com/miekg/dns#Client)。

# TODO

- [ ] 命令行参数配置
- [ ] 外部可用 IP 缓存文件
- [ ] 多上游无污染 DNS
- [ ] 直接拦截代理而不使用本地 Pipe
- [ ] 待 ESNI 普及后尝试实现
- [ ] 待 QUIC 普及后尝试实现