
---

# MosDNS 配置文件说明文档

本仓库提供了一份基于 **MosDNS (v5)** 的高性能、防污染、带有智能分流与缓存机制的 DNS 配置文件。

该配置通过 `Socks5` 代理请求海外 DNS，利用 `ECS (EDNS Client Subnet)` 优化国内解析，并内置了 `Fallback`（主备切换）机制，旨在提供快速、准确且无 DNS 泄露的解析体验。

## 📁 核心架构

配置的总体逻辑基于 MosDNS 的**插件化流水线（Sequence）**设计。请求的流转主要分为以下三个阶段：

1. **Pre_Sequence (预处理)**：拦截非法请求、过滤广告域名、处理局域网/内网请求。
2. **Main_Sequence (主处理)**：查询缓存 -> 匹配国内/国外域名列表 -> 分发至对应的上游 DNS 组。
3. **Query Sequences (具体查询)**：执行 ECS 处理、并发请求、结果过滤。

## 🛠️ 核心模块解析

### 1. 基础服务与日志

* **HTTP API**：监听 `127.0.0.1:8338`（用于查看运行状态和 Metrics）。
* **DNS 服务**：同时开启 UDP 和 TCP 监听 `127.0.0.1:5335`。
* **日志级别**：`warn`（关闭了生产环境模式以输出更详细警告）。

### 2. 上游 DNS 分组 (Upstreams)

由于网络环境限制，海外 DNS 均配置了 `socks5: 127.0.0.1:7891` 进行代理查询，并利用 `dial_addr` 绑定特定 IP（避免代理过程中的二次 DNS 解析）。

* **海外组 (需代理)**：
  * `google`、`cloudflare`、`quad9`：常用海外公共 DNS。
  * `nextdns`：需自行注册配置 ID（目前配置中为占位符 `xxxxxx`）。
  * `dns_nocn`：`fallback` 插件组，主用 NextDNS，备用 Quad9，阈值 600ms。
* **国内组 (直连)**：
  * `ali` (阿里)、`dnspod` (腾讯)：内置并发查询，部分启用了 HTTP/3 或 TLS 流水线。
  * `dns_cn`：`fallback` 插件组，主用阿里，备用 DNSPod，阈值 600ms。
* **本地组**：
  * `local`：使用 `udp://192.168.1.1:53`（通常为路由器网关）。

### 3. 缓存与 ECS (EDNS Client Subnet)

* **缓存机制**：
  * `cache_lan`：针对局域网，容量 8192，懒缓存 1 天。
  * `cache_wan`：针对公网，容量 131072，懒缓存 2 小时。
* **ECS 处理**：
  * `ecs_cn`：查询国内域名时，附带 `183.2.172.177` 的 ECS，帮助国内 CDN 精准调度。
  * `no_ecs`：查询海外域名时，**不发送** ECS，防止隐私泄露。

### 4. 规则集 (Data Sets)

配置引用了来自 Nix Store 的规则文件（需根据实际路径调整）：

* **IP 集**：`geoip_private` (私有地址), `geoip_cn` (国内 IP)。
* **域名集**：`geosite_cn` (国内域名), `geosite_gfw` (被墙域名), `geosite_location-!cn` (非中国大陆域名), `geosite_ads-all` (广告域名)。

---

## 🚀 解析逻辑流 (核心策略)

### 第一阶段：预处理 (`pre_sequence`)

1. 拒绝 `qtype 65` (HTTPS 记录) 和包含 `keyword::` 的空域名。
2. 如果匹配 `geosite_ads-all`，直接拦截（返回 NXDOMAIN）。
3. 如果查询内网 IP (PTR 记录) 或特定类型 (12, 255)，转交 `query_other` 处理。
4. 其他请求进入下一阶段，并优先检查是否在预处理阶段已有结果（如命中缓存）。

### 第二阶段：主路由 (`main_sequence`)

1. 先查 `cache_wan`。
2. 匹配 `geosite_gfw` 或 `geosite_location-!cn`（被墙或海外域名）-> 交给 **`query_nocn`**。
3. 匹配 `geosite_cn`（国内域名）-> 交给 **`query_cn`**。
4. 未匹配到任何规则 -> 交给 **`query_fallback`** (防污染兜底逻辑)。

### 第三阶段：查询执行

* **`query_cn`**：加入国内 ECS -> 查询国内组 (`dns_cn_seq`) -> **过滤污染 IP** (drop_resp 包含常见 DNS 劫持 IP 列表)。
* **`query_nocn`**：不使用 ECS -> 查询海外组 (`dns_nocn_seq`) -> **过滤** 包含国内 IP 的返回结果（防止 DNS 投毒）。
* **`query_fallback`**：不使用 ECS -> 查询海外组 (`fallback_seq`) -> **过滤** 包含国内 IP 的返回结果。

---

## ⚠️ 使用前必读 (注意事项)

1. **代理配置**：配置中假定本地 `127.0.0.1:7891` 运行着 Socks5 代理。请根据实际情况修改。
2. **NextDNS 配置**：`nextdns` 标签下需要替换为您自己的 NextDNS 配置 ID (`xxxxxx`) 及对应的 IP。
3. **规则路径**：所有的 `files` 路径（如 `/nix/store/...`）是 NixOS 的特定路径。如果在其他系统上运行，请修改为实际的 `geosite` 和 `geoip` 文件路径。
4. **本地上游**：`udp://192.168.1.1:53` 需修改为您实际的路由器 IP。
5. **防污染 IP 列表**：`drop_resp` 中包含了一长串硬编码的 IP，这些是已知的运营商 DNS 劫持 IP，可根据实际情况维护。

## 🔧 进阶优化建议

* **并发控制**：可以通过调整各个 `forward` 插件中的 `concurrent` 值来平衡解析速度和资源消耗。
* **Metrics 监控**：如果不需要性能指标监控，可以移除 `sequence` 里的 `metrics_collector metrics` 来节省少量性能。
* **广告过滤**：当前 `geosite_ads-all` 是被直接 `reject 3` 的，如果发现某些正常服务被误杀，可以尝试改为 `reject 2` (REFUSED) 或者移除该规则。
