
# proxy_config

个人维护的代理客户端配置文件集合，涵盖多种主流代理工具。

## 目录结构

| 目录 | 说明 |
| ------ | ------ |
| `clash` | Clash / Mihomo 核心配置 |
| `dae` | dae 配置 |
| `loon` | Loon 配置 |
| `mosdns` | MosDNS 配置 |
| `quantumult_x` | Quantumult X 配置 |
| `stash` | Stash 配置 |
| `surge` | Surge 配置（含 Smart 配置） |

## 架构说明：dae + mihomo + mosdns

本仓库中的 `dae`、`clash`（即 mihomo）、`mosdns` 组合构成了一套高性能、防污染的分层代理与 DNS 架构：

1. **MosDNS（DNS 分流与防污染）**：作为本地 DNS 服务，负责国内外域名的智能分流。国内域名走直连 DNS（如阿里、腾讯），海外域名通过代理隧道查询，并内置 ECS 优化与缓存机制，确保解析快速且无污染。
2. **Mihomo（代理核心）**：作为实际的代理处理引擎，接收由 dae 透明转发过来的流量。负责规则的最终匹配与节点负载均衡。
3. **dae（透明代理入口）**：基于 eBPF 技术的高性能透明代理工具。在 Linux 系统底层直接接管流量，具有极低的性能损耗。它将需要代理的请求转发给 Mihomo，并根据规则将 DNS 请求指向本地的 MosDNS。

> **数据流向**：系统/局域网流量 -> `dae`（eBPF 拦截） -> `MosDNS`（DNS 解析分流） -> `Mihomo`（代理节点转发）。

> **📖 详细部署与 NixOS 配置参考**：关于此架构在 NixOS 下的具体实现，请参阅 [NixOS Network Manager 配置说明](https://github.com/mochouaaaaa/nix-config/blob/flake-parts/modules/nixos/services/network-manager/README.md)。

## 说明

- **Surge** 和 **Quantumult X** 为独立维护的稳定配置，持续调优中。
- 其余配置供参考使用，更新频率不一。

## 使用

克隆仓库后，按客户端类型选择对应目录中的配置文件导入即可：

```bash
git clone https://github.com/mochouaaaaa/proxy_config.git
```

请根据自身网络环境调整规则、策略组和上游设置。

## 注意

- 部分配置可能引用证书或外部资源，使用前请自行审阅。
- 使用本仓库配置产生的任何后果由使用者自行承担。

---
