# Shadowrocket 中国大陆直连 / 国外与未知代理 / DNS 隐私兼容版

这是一个只使用 Shadowrocket 原生配置语法与 Shadowrocket 原生远程规则的无策略组配置。

## 一键导入

长期 Raw 链接：

```text
https://raw.githubusercontent.com/wtang2217-eng/shadowrocket-lazy-dns-protected/main/lazy_dns_protected.conf
```

本次热修缓存刷新链接：

```text
https://raw.githubusercontent.com/wtang2217-eng/shadowrocket-lazy-dns-protected/main/lazy_dns_protected.conf?v=20260815-hotfix3
```

如果上一版导入后所有节点都显示超时：

1. 先完全关闭 Shadowrocket VPN。
2. 删除手机中那份旧配置，再用上面的缓存刷新链接重新导入。
3. 首页选择一个原本可用的节点。
4. 对新配置执行“使用配置 / 编译配置”。
5. 把“全局路由”设为“配置”，再重新连接。
6. 建议使用 Shadowrocket 2.2.64 或更高版本；较旧版本不完整支持这里的 DNS 代理/ECS 语法。

## 本次热修

全节点超时与上一版同时引入的四项高风险改动高度相关。本版已经：

- 删除覆盖全部 IPv4 的 `tun-included-routes = 0.0.0.0/1,128.0.0.0/1`。
- 不再把 Fake-IP 保留段 `198.18.0.0/15` 排除在 TUN 外。
- 删除可能误伤域名型/IPv6 节点的 `::/0 REJECT`。
- 把节点域名引导恢复为此前可连接版本的 `tls://1.1.1.1:853,tls://223.5.5.5:853`。
- 删除 ChinaMax 超大表，降低首次下载、编译和 Network Extension 内存压力。

## 通用分流，不给检测站“写答案”

配置文件中没有 BrowserLeaks、DNSLeakTest、IPLeak、DNSCheck、Sukka、net.vin 或其他检测站的域名特例。它们与普通网站经过完全相同的规则链：

1. 局域网地址直连。
2. Blackmatrix7 的 Shadowrocket 原生 AI/Global 国外规则走 `PROXY`。
3. ChinaIPsBGP 与 `GEOIP,CN` 判定为中国大陆 IP 的流量走 `DIRECT`。
4. 其他国外、未知域名和未知 IP 由 `FINAL,PROXY` 接管。

因此，BrowserLeaks 这类国外网站显示代理出口，是“国外与未知走代理”这条通用策略自然得到的结果，并非为检测站单独强制。net.vin 页面本身如果解析到大陆 IP，会自然直连；它发出的腾讯、字节、AI 等探测请求仍分别经过同一套规则判断。

这里不再使用容易把国外网站误收为“中国”的 China/ChinaMax 域名聚合表。未知域名会通过代理侧加密 DNS 取得带固定中国 ECS 的结果，再由 `GEOIP,CN` 判断：大陆地址直连，非大陆地址继续代理。

## DNS 模型

- 普通域名与直连域名使用 Google DoH，DoH 请求本身通过首页当前选择的代理节点。
- 固定 ECS `120.76.0.0/14` 只用于获得大陆 CDN，不是手机的真实 IP，也不会随你以后更换所在地而改变。
- 主 DNS、直连 DNS 与备用 DNS 均不使用 `system` 或明文 DNS。
- `hijack-dns = :53` 捕获进入隧道的普通 TCP/UDP 53。
- 节点建立之前无法使用该节点解析它自己，所以节点域名单独使用 IP 字面量 DoT 引导；这只涉及节点主机名，不包含日常访问的国外业务域名。
- `ipv6 = false`、`prefer-ipv6 = false` 关闭 Shadowrocket 的普通 IPv6 业务流量。

## 如何正确看检测结果

BrowserLeaks 顶部的 “Your IP” 是该网页看到的 HTTP 出口，不是 DNS。国外测试页在“配置”路由下应显示当前选择节点的出口，不能显示手机运营商公网 IP。

DNS 表显示的是递归解析器出口。出现多个 Google/Cloudflare Anycast 地址是正常的；真正需要警惕的是中国电信、联通、移动、家庭路由器或当前所在地 ISP 的递归 DNS。

请分别在 Wi-Fi 与蜂窝网络测试：

- https://browserleaks.com/dns
- https://www.dnsleaktest.com/
- https://dnscheck.tools/
- https://ipleak.net/
- https://test-ipv6.com/
- https://ip.skk.moe/
- https://net.vin/

同时在 Shadowrocket“数据”中核对每个请求实际命中的规则与策略。测试站旗帜或商业 GeoIP 可能误判位置，连接日志才是分流依据。

## 无法由通用配置消除的边界

如果订阅节点地址本身是域名，建立代理前必须先解析一次节点域名。DoT 会加密这段传输，但对应解析器仍能看到节点主机名与请求源 IP。要连这一次也完全消除，节点必须直接使用 IPv4，或为每个节点域名维护稳定的 IPv4 Host 映射；通用配置无法预先知道订阅商的节点地址。

应用自带的 DoH/DoT/DoQ 属于普通 HTTPS/TLS/QUIC 流量，`:53` 无法识别其内部查询；它们仍按上述通用分流规则走 DIRECT 或 PROXY。iOS 上其他 VPN/DNS 描述文件、Private Relay，以及 Shadowrocket 未连接时的流量也不受本配置控制。

## 数据与兼容性来源

- Shadowrocket 配置语义：https://github.com/LOWERTOP/Shadowrocket/wiki/
- Blackmatrix7 Shadowrocket 原生规则：https://github.com/blackmatrix7/ios_rule_script/tree/master/rule/Shadowrocket
- ChinaIPsBGP：https://github.com/blackmatrix7/ios_rule_script/tree/master/rule/Shadowrocket/ChinaIPsBGP
- Google Public DNS DoH：https://developers.google.com/speed/public-dns/docs/doh/
- Cloudflare DNS over TLS：https://developers.cloudflare.com/1.1.1.1/encryption/dns-over-tls/