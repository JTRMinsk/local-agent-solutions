# CC Switch: Claude Code Protocol Translator

## 📝 问题场景
Claude Code 默认遵循 Anthropic 的 API 标准协议。当你尝试使用第三方 Provider（如 SiliconFlow 提供的 GLM-5.2）时，会出现以下问题：
- **协议冲突**：第三方模型虽然支持 OpenAI 格式，但在 `tools` 结构、`system` 提示词位置等细节字段上与 Anthropic 标准存在细微差异。
- **调用失败**：直接调用会导致 API 返回格式错误或无法识别参数，导致 Claude Code 无法正常工作。

## ✅ 解决方案 (本质：协议转换器 Protocol Translator)

`cc-switch` 作为一个中间层代理，通过以下三个核心功能实现兼容：

### 1. 拦截 (Intercept)
拦截 Claude Code 发出的所有原生 API 请求。

### 2. 重写 (Rewrite)
将 Anthropic 标准格式的请求体（Request Body）转换为标准的 **OpenAI Chat Completion** 格式。
- **核心配置**：`apiFormat: openai_chat`

### 3. 路由 (Route)
将重写后的请求转发至目标 Provider（如 SiliconFlow 或其他支持 OpenAI 协议的服务）。

## 🚀 意义
通过这种方式，你可以实现“**假装自己是 Claude**”的效果，从而驱动任何支持 OpenAI 协议的第三方模型，完美解决工具调用和协议不兼容的问题。
