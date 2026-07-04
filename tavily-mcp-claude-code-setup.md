# Claude Code + Tavily MCP：让第三方 Provider 也能联网搜索

> 日期：2026-07-04
> 环境：Claude Code → CC Switch proxy (15721) → SiliconFlow GLM-5.2

## 背景：为什么内置搜索用不了？

Claude Code 自带两个联网工具：**WebSearch** 和 **WebFetch**。

问题出在 WebSearch——它不是在本地跑的，而是把搜索请求发到 **Anthropic 服务器**，由 Anthropic 替你执行搜索，再把结果塞回对话上下文。

```
内置 WebSearch 链路：
Claude Code → Anthropic 服务器（执行搜索）→ 返回结果
              ↑
        这一步需要 Anthropic API
```

但我们把 Claude Code 接到了 SiliconFlow GLM-5.2（通过 CC Switch 做格式转换），请求根本到不了 Anthropic 服务器。GLM-5.2 收到 web_search 工具调用，但它后面没有搜索引擎，搜索就失败了。

**这不是 bug，是架构限制**：服务端工具只有连着服务端才工作。所有用第三方 provider（SiliconFlow / Z.ai / DeepSeek / OpenRouter）跑 Claude Code 的人都会遇到这个问题。

## 解法：换成本地 MCP 搜索工具

MCP（Model Context Protocol）工具调用不走 Anthropic 服务端。Claude Code 作为 MCP client，通过本地 stdio 与 MCP server 通信，搜索由 MCP server 在本地执行，结果作为 tool_result 注入对话上下文。

```
MCP 搜索链路：
Claude Code → Tavily MCP server（本地 stdio 进程）
                    ↓
              Tavily API（执行搜索）
                    ↓
            结果作为 tool_result 返回
                    ↓
         CC Switch 格式转换 → GLM-5.2 读取结果
```

整条链路不经过 Anthropic，模型只需要具备 function calling 能力——GLM-5.2 已实测支持。

## 关键改动

### 1. 禁用内置 WebSearch / WebFetch

文件：`~/.claude/settings.json`

```json
{
  "permissions": {
    "deny": ["WebSearch", "WebFetch"]
  }
}
```

**为什么连 WebFetch 也禁？** WebFetch 抓网页后，用 Anthropic 服务端的 Haiku 模型做二次摘要。第三方 provider 下 Haiku 不可靠，搜索+抓取全交给 Tavily MCP 一步到位。

**如果不禁会怎样？** Claude Code 仍会注册内置 WebSearch 工具，模型可能优先调它（然后失败），而不是去用 MCP 的 `tavily_search`。deny 掉后模型只能走 MCP 这一条路。

### 2. 添加 Tavily MCP server

```bash
claude mcp add tavily -s user \
  -e TAVILY_API_KEY=tvly-dev-xxxx \
  -- npx -y tavily-mcp
```

写入 `~/.claude.json`，配置如下：

```json
{
  "mcpServers": {
    "tavily": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "tavily-mcp"],
      "env": {
        "TAVILY_API_KEY": "tvly-dev-xxxx"
      }
    }
  }
}
```

**为什么用 Tavily？**
- 已有 API key（Hermes 的 web search 也在用 Tavily）
- 有成熟的 MCP server（`tavily-mcp` npm 包，v0.2.20）
- 提供 search / extract / crawl 三个工具
- npm 国内可直连（npmmirror 0.2s）

**为什么用 stdio 不用远程 MCP？** 本地进程更稳定，不依赖 Tavily 远程 MCP 服务器可用性。首次 `npx -y tavily-mcp` 有约 5 秒下载延迟，之后有缓存。

### 3. 重启 Claude Code

MCP server 在启动时连接。`claude mcp list` 可确认连接状态。

## 前置验证

动手前实测了以下环节，确保链路可行：

| 验证项 | 结果 | 说明 |
|---|---|---|
| GLM-5.2 function calling | ✅ | 正确返回 `tool_calls` |
| 工具结果往返 | ✅ | 模拟 tool_result 返回，GLM-5.2 正确读取并生成最终回答 |
| npx / node | ✅ | node v22.22.3, npx 10.9.8 |
| npm 国内访问 | ✅ | npmmirror 0.2s 直连 |
| tavily-mcp 包 | ✅ | v0.2.20，提供 search/extract/crawl |
| Tavily API key | ✅ | 已有（`~/.hermes/.env` 中 `TAVILY_API_KEY`）|

## 回退方法

改动前已备份：

```bash
# 备份文件
~/.claude.json.bak.20260704233121
~/.claude/settings.json.bak.20260704233121

# 一键回退
cp ~/.claude/settings.json.bak.20260704233121 ~/.claude/settings.json
cp ~/.claude.json.bak.20260704233121 ~/.claude.json
```

回退后重启 Claude Code 即恢复原状。

## 涉及文件

| 文件 | 改动 |
|---|---|
| `~/.claude/settings.json` | 加 `permissions.deny: ["WebSearch", "WebFetch"]` |
| `~/.claude.json` | 加 `mcpServers.tavily` 配置 |
