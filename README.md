# Shadowrocket 中国直连 / 境外与未知代理 / DNS 隐私强化版

这是为 Shadowrocket 原生配置语法制作的无策略组版本。它不使用 Clash/Mihomo、Quantumult X、Surge 或 V2Ray 的规则格式。

## 一键导入

长期更新链接：

```text
https://raw.githubusercontent.com/wtang2217-eng/shadowrocket-lazy-dns-protected/main/lazy_dns_protected.conf
```

本次缓存刷新链接：

```text
https://raw.githubusercontent.com/wtang2217-eng/shadowrocket-lazy-dns-protected/main/lazy_dns_protected.conf?v=20260815-2
```

导入后：

1. Shadowrocket 首页选择一个可用节点。
2. “全局路由”选择“配置”，不能选“直连”或“代理”。
3. 配置页对该文件执行“使用配置 / 编译配置”。
4. 断开后重新连接；必要时删除旧配置再用缓存刷新链接导入。
5. 规则集自动更新时需要能访问 GitHub Raw。

## 路由模型

优先级从上到下：

1. 局域网地址直连。
2. IPv6 业务流量拒绝。
3. DNS/IP/分流检测、公共 DoH、AI 与 Global 境外规则代理。
4. ChinaMaxNoIP 主表和 DOMAIN-SET 域名表直连。
5. ChinaIPsBGP 与 GeoIP CN 的硬编码大陆 IP 直连。
6. `FINAL,PROXY`：未知域名与未知 IP 一律使用首页当前选择的节点。

旧配置的核心错误是只引用了不完整的中国列表，并把宽泛 Global 放在错误的组合中。更关键的是，Blackmatrix7 的大型 Shadowrocket 列表将大部分 `DOMAIN-SUFFIX` 放在单独的 `*_Domain.list`；漏掉 DOMAIN-SET 会导致腾讯、字节等大量域名走到 FINAL 代理。

## DNS 模型

- PROXY 域名由代理服务器远程解析，手机不为这些业务域名做本地解析。
- DIRECT 域名使用 Google DoH，但 DoH 请求本身经首页当前节点，并强制中国 ECS `120.76.0.0/14`。
- ECS 是固定的中国网段，只用于取得大陆 CDN，不包含手机真实 IP 或真实所在地。
- 主 DNS 与备用 DNS 都不含 `system` 或明文 DNS，失败即关闭，不回退运营商 DNS。
- `hijack-dns = :53` 捕获进入隧道的普通 TCP/UDP 53。
- 节点域名存在“先有节点还是先有代理”的启动循环，因此单独使用阿里 `223.5.5.5` 与 DNSPod `1.12.12.12` 的 IP 字面量 DoH；ISP 看不到节点域名明文。

已于 2026-08-15 实测同一中国 ECS：

- `perfops.byte-test.com` 返回中国大陆 `124.225.* / 125.94.* / 59.38.*` 等地址。
- `r.inews.qq.com` 返回中国大陆 `183.47.104.164 / 183.47.104.207`。

## 正确理解 BrowserLeaks

页面顶部的 “Your IP” 是网站看到的 HTTP 出口。境外测试页应显示当前代理出口，不能显示手机运营商公网 IP。

DNS 表会列出递归解析器出口。看到 Cloudflare 或 Google 的多个 Anycast 地址并不等于“多个节点泄漏”；真正异常的是出现中国电信、联通、移动、家庭路由器或当前网络运营商的递归 DNS。

## 真机验收

分别在 Wi-Fi 与蜂窝网络测试：

- https://browserleaks.com/dns
- https://www.dnsleaktest.com/
- https://dnscheck.tools/
- https://ipleak.net/
- https://test-ipv6.com/
- https://ip.skk.moe/
- https://net.vin/

预期：

- BrowserLeaks 顶部 IP 是所选代理出口。
- DNS 列表只有代理侧/配置侧解析器，不出现当前 ISP DNS。
- test-ipv6.com 不显示公网 IPv6。
- net.vin 的字节与腾讯国内探针走 DIRECT；AI 与境外探针走 PROXY。

定位时在 Shadowrocket“数据”中检查：

- 代理日志：`browserleaks.com` 应命中 PROXY；`perfops.byte-test.com` 与 `r.inews.qq.com` 应命中 DIRECT。
- DNS 日志：不应出现 `system`、家庭路由器、当前 ISP 的 DNS 上游。

## 不能被通用配置消除的启动边界

如果节点地址本身是域名，建立代理之前必须先解析一次该节点域名。这里已用国内加密 DoH 保护传输，但阿里/DNSPod 仍能看到该节点域名与请求源 IP。要连这一次也完全消除，订阅节点地址必须直接使用 IPv4，或为每个节点域名维护稳定的 IPv4 Host 映射；通用配置无法预先知道订阅商的节点域名和 IP。

同样，`ipv6 = false` 不一定阻止“节点域名”自身使用 AAAA。若要绝对禁止节点 IPv6，节点地址也应使用 IPv4 字面量。

## 数据与兼容性来源

- Shadowrocket 社区手册（配置语义）：https://github.com/LOWERTOP/Shadowrocket/wiki/
- Blackmatrix7 Shadowrocket 原生规则：https://github.com/blackmatrix7/ios_rule_script/tree/master/rule/Shadowrocket
- ChinaMaxNoIP 使用说明：https://github.com/blackmatrix7/ios_rule_script/blob/master/rule/Shadowrocket/ChinaMaxNoIP/README.md
- ChinaIPsBGP 使用说明：https://github.com/blackmatrix7/ios_rule_script/blob/master/rule/Shadowrocket/ChinaIPsBGP/README.md
- Google Public DNS DoH：https://developers.google.com/speed/public-dns/docs/doh/
