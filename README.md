# Shadowrocket 中国大陆直连 / 国外与未知代理 / 低延迟 DNS 隐私版

这是一个只使用 Shadowrocket 原生配置语法与 Shadowrocket 原生远程规则的无策略组配置。目标是：

- 常用国内服务保持直连和本地 CDN 性能。
- 国外与未知流量使用首页当前选中的节点。
- 国外与未知域名不交给本地系统 DNS 或运营商 DNS。
- 不给任何检测站添加专用规则。

## 一键导入

长期 Raw 链接：

~~~text
https://raw.githubusercontent.com/wtang2217-eng/shadowrocket-lazy-dns-protected/main/lazy_dns_protected.conf
~~~

本次低延迟版缓存刷新链接：

~~~text
https://raw.githubusercontent.com/wtang2217-eng/shadowrocket-lazy-dns-protected/main/lazy_dns_protected.conf?v=20260815-balanced1
~~~

建议先完全关闭 Shadowrocket VPN、删除旧配置，再使用缓存刷新链接导入。随后：

1. 首页选择一个可用节点。
2. 对新文件执行“使用配置 / 编译配置”。
3. 将“全局路由”设为“配置”。
4. 重新连接。
5. 建议使用 Shadowrocket 2.2.64 或更高版本。
6. 若节点支持 UDP，请在 Shadowrocket 设置和节点详情中启用 UDP 转发，才能使用代理内 QUIC/HTTP3。

## 低延迟 split DNS

DNS 被拆为三个互不混淆的路径：

### 高置信国内域名

.cn 和阿里、腾讯、字节、百度、国内媒体、网易、华为、小米、京东、美团、拼多多、快手、滴滴、爱奇艺、优酷等高置信国内规则，使用：

~~~ini
direct-dns-server = https://dns.alidns.com/dns-query#no-h3
~~~

它通过 HTTPS 加密，运营商看不到查询内容；阿里 DNS 会看到真实出口 IP 和这些国内域名。固定 dns.alidns.com = 223.5.5.5，避免解析加密 DNS 端点本身时调用系统 DNS。

### 国外与未知域名

明确国外域名由代理端远程解析。未知域名需要判断 IP 地区时，查询通过当前选择节点发送给 Google DoH：

~~~ini
dns-server = https://dns.google/dns-query#proxy=proxy&ecs=120.76.0.0/14&ecs-override=true
~~~

固定 ECS 是用于大陆 CDN 调度的虚拟中国网段，不是手机真实 IP，也不会随手机所在地变化。查询失败时只允许通过当前节点使用 Cloudflare DoH：

~~~ini
fallback-dns-server = https://cloudflare-dns.com/dns-query#proxy
~~~

配置不含 system 或明文 DNS 回退。

### 节点启动 DNS

域名型节点必须在代理建立前解析自身，因此继续使用此前真机可连接的 IP 字面量 DoT：

~~~ini
proxy-dns-server = tls://1.1.1.1:853,tls://223.5.5.5:853
~~~

这段链路加密，运营商看不到节点域名；Cloudflare/阿里解析器仍会看到节点域名和手机出口 IP。这是域名型节点的启动边界，使用 IPv4 地址形式的节点才能彻底消除。

## 通用分流顺序

配置中没有 BrowserLeaks、DNSLeakTest、IPLeak、DNSCheck、Sukka、net.vin 等检测站域名。所有网站经过同一条规则链：

1. 局域网地址直连。
2. 明确国外 AI 与 Global 规则走 PROXY。
3. 高置信国内域名走加密国内 DNS 后 DIRECT。
4. 硬编码大陆 IP 走 DIRECT，且不触发域名解析。
5. 未知域名通过代理 DoH 解析；结果为大陆 IP 才由 GEOIP,CN 直连。
6. 其他国外、未知域名和未知 IP 由 FINAL,PROXY 接管。

因此，国外测试页显示代理出口，是通用“国外/未知代理”策略的自然结果，并非提前给检测站写答案。

## QUIC 与速度

本版使用：

~~~ini
block-quic = always-allow
udp-policy-not-supported-behaviour = REJECT
~~~

DIRECT 和 PROXY 都可以使用 QUIC/HTTP3。节点支持 UDP 且线路质量良好时，国外网页和视频的握手延迟通常更低。

如果节点不支持 UDP，Shadowrocket 会拒绝该 UDP 包，绝不自动 DIRECT。浏览器随后自行改用 TCP/HTTP2 时，会再次命中相同的 PROXY 规则，因此不会因为回退而暴露真实 IP。

## 正确理解检测结果

BrowserLeaks 顶部 “Your IP” 是网页看到的 HTTP 出口。国外测试页应显示当前节点出口，不能显示手机运营商公网 IP。

DNS 表显示递归解析器出口，而不是手机里存在多少个节点：

- 国外检测应看到代理侧 Google、Cloudflare 或节点提供的解析器。
- 不应看到当前所在地的运营商、家庭路由器或系统 DNS。
- 多个 Google/Cloudflare Anycast 地址属于正常现象。
- 高置信国内网站使用阿里 DoH 是本配置主动设置的国内加密 DNS，不代表国外业务域名泄漏给阿里。

分别在 Wi-Fi 与蜂窝网络测试：

- https://browserleaks.com/dns
- https://www.dnsleaktest.com/
- https://dnscheck.tools/
- https://ipleak.net/
- https://test-ipv6.com/
- https://ip.skk.moe/
- https://net.vin/

同时在 Shadowrocket“数据”中查看每个请求实际命中的规则。商业 GeoIP 的国家旗帜可能误判，连接日志才是分流依据。

## 无法由通用配置完全覆盖的边界

- 国内网站既然 DIRECT，就会看到手机真实公网 IP；这是直连的定义。
- 应用自带 DoH/DoT/DoQ 属于普通 HTTPS/TLS/QUIC 流量，端口 53 劫持无法读取其内部查询。国外解析器端点通常会落到 Global/FINAL 并走代理；硬编码的国内 DNS IP 仍可能按大陆 IP 直连。
- iOS 上的其他 VPN/DNS 描述文件、Private Relay，以及 Shadowrocket 未连接时的流量不受此配置控制。
- 任何规则库都不能从域名本身百分之百证明服务“国籍”。本配置让国外规则优先，并只为高置信国内服务启用本地 DoH；其余一律走代理 DoH 后再判断。

## 数据与兼容性来源

- Shadowrocket 配置语义：https://github.com/LOWERTOP/Shadowrocket/wiki/
- Blackmatrix7 Shadowrocket 原生规则：https://github.com/blackmatrix7/ios_rule_script/tree/master/rule/Shadowrocket
- ChinaIPsBGP：https://github.com/blackmatrix7/ios_rule_script/tree/master/rule/Shadowrocket/ChinaIPsBGP
- Google Public DNS DoH：https://developers.google.com/speed/public-dns/docs/doh/
- Cloudflare DNS over TLS：https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-tls/