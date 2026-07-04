# GitHub Network Fallback (SNI Blocking & Large File Issues)

## 📝 问题场景
在国内 ISP 环境下，直接使用 `git clone` 或 `gh` 命令进行操作时，经常会遇到以下问题：
- **SNI 阻断**：在 TLS 握手阶段被运营商拦截，导致连接卡死。
- **大文件传输超时**：在使用代理（如 `mihomo`）时，由于带宽抢占或代理链路稳定性问题，在大文件传输过程中容易导致连接超时断开。

## ✅ 解决方案

### 1. 协议分流 (Protocol Split)
对于体积较小的仓库，优先尝试使用国内镜像源进行克隆，以规避 TLS 阻断：
```bash
git clone https://gh.ddlc.top/<username>/<repo>.git
```

### 2. 代理策略 (Proxy Strategy)
针对大文件下载任务，不依赖全局代理，而是通过 `curl` 手动指定本地代理，并限制下载速率，防止因带宽过载导致的连接中断：
```bash
curl --proxy http://127.0.0.1:7890 --limit-rate 5M -O <URL>
```

### 3. 身份验证优化 (Auth Optimization)
为了绕过复杂的交互式登录过程，并将身份验证直接集成在指令中，建议将 **PAT (Personal Access Token)** 直接嵌入到 Git Remote URL 中：
```bash
git clone https://<your_token>@github.com/<username>/<repo>.git
```

## 🛠️ 验证方法
- 使用 `git clone` 镜像源观察是否能快速完成握手。
- 使用 `curl --proxy` 观察大文件下载的稳定性。
