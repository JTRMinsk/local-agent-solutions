# Hermes Gateway Proxy Optimization (NO_PROXY)

## 📝 问题场景
Hermes Gateway 在后台运行过程中，其代理行为可能导致以下问题：
- **访问国内 API 极慢/失败**：如果 Gateway 走全局代理（如 `mihomo`），访问国内服务（如 SiliconFlow、飞书、智谱 AI）时，会因为经过海外中转导致响应延迟剧增，甚至因 SSL 握手问题导致连接失败。
- **无法访问本地服务**：如果配置不当，Gateway 可能会尝试通过代理去访问 `127.0.0.1` 上的本地组件（如 `cc-switch`），导致连接无法建立。

## ✅ 解决方案

### 1. 配置劫持 (Config Hijacking)
通过修改 `hermes-gateway.service` 的 Systemd 服务配置文件，利用 `Environment="NO_PROXY=..."` 指令显式定义白名单，确保特定流量不经过代理。

### 2. 白名单配置内容
建议将以下域名和地址加入 `NO_PROXY`：
- **SiliconFlow**: `api.siliconflow.cn`
- **飞书 (Lark)**: `open.feishu.cn`
- **智谱 AI**: `api.zhipuai.com`
- **本地服务**: `127.0.0.1`, `localhost`

## 🚀 效果
实现“**国内 API 直连，国外 API 走代理**”的自动化路由。既保证了对 Anthropic/OpenAI 等海外服务的访问能力，又确保了国内服务的高速、稳定直连。
