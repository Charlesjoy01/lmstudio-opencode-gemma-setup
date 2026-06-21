# 🚀 LM Studio + OpenCode + Gemma Setup

> 基于 LM Studio + OpenCode + Gemma 4 (12B) 的本地 AI 编程助手的潜在问题修复指南。

## 📖 简介

本项目记录了在本地消费级硬件（如 24GB M系列 MacBook Air）上，部署开源大模型（Gemma 4 12B）并接入终端 AI 编程助手（OpenCode）的完整实战过程。  
本次部署的模型：
[yuxinlu1/gemma-4-12B-agentic-fable5-composer2.5-v2-3.5x-tau2-GGUF](https://huggingface.co/yuxinlu1/gemma-4-12B-agentic-fable5-composer2.5-v2-3.5x-tau2-GGUF)

这个项目解决了接入过程中最常见的两个"拦路虎"：
1. **Jinja 模板渲染崩溃**（`UndefinedValue` 报错）。
2. **上下文溢出**（`n_keep >= n_ctx` 报错）。

无论你是 AI 产品实习生、独立开发者，还是想在本地跑 Vibe Coding 的极客，这份手册都能帮你跳过踩坑阶段，直接获得丝滑的本地 AI 编程体验。

## 💻 硬件与模型基准

*   **硬件基准**：Apple MacBook Air (M系列芯片, 24GB 统一内存)
*   **模型选择**：Gemma 4 12B (推荐 Q4_K_M 或 Q5_K_M 量化版 GGUF)
*   **软件环境**：LM Studio (0.3.x+), OpenCode

### ⚖️ 内存与 Context Window 的 Trade-off
在 24GB 内存下，12B Q4 模型权重约占 8-9GB，系统占用约 6GB。剩余约 9GB 用于 KV Cache。
*   **8K Context**：极致流畅，适合单文件代码生成。
*   **16K Context (推荐)**：平衡之选，可容纳完整的 System Prompt + Tool Definitions + 多轮对话。
*   **32K Context**：适合跨文件重构，但需注意 MacBook Air 无风扇设计可能导致的降频。

---

## 🐛 报错 1：Jinja 模板渲染崩溃

### 完整报错信息

```
Error rendering prompt with jinja template: "Cannot call something that is not a function: got UndefinedValue".

This is usually an issue with the model's prompt template. If you are using a popular model, you can try to search the model under lmstudio-community, which will have fixed prompt templates. If you cannot find one, you are welcome to post this issue to our discord or issue tracker on GitHub. Alternatively, if you know how to write jinja templates, you can override the prompt template in My Models > model settings > Prompt Template.
```

### 原因分析

社区转换的 Gemma 4 GGUF 模型，其自带的 `chat_template` 缺失了处理 Tool Calling 参数类型的 `format_type_argument` 宏。当 OpenCode 注入工具定义（JSON Schema）时，Jinja 渲染器遇到未知函数调用直接崩溃，抛出 `UndefinedValue` 错误。

### 解决方案

1. 打开 LM Studio，进入模型的 **Prompt Template** 设置
2. 将 Prompt Format 设为 `Custom`
3. 删除原有模板内容，粘贴本项目提供的修复版模板 👉 [`templates/gemma4_full_feature.jinja`](./templates/gemma4_full_feature.jinja)
4. 重新加载模型

**修复原理**：补全了缺失的 `format_type_argument` 宏，使其能正确处理 Tool Calling 的 JSON Schema 参数类型，包括 `string`、`array`、`object` 等嵌套结构。

---

## 🐛 报错 2：上下文溢出

### 完整报错信息

```
The number of tokens to keep from the initial prompt is greater than the context length (n_keep: 7319 >= n_ctx: 4096). Try to load the model with a larger context length, or provide a shorter input.
```

### 原因分析

开启 Tool Calling 后，OpenCode 会注入大量工具的 JSON Schema 定义。LM Studio 默认 Context Length 为 4096，而 System Prompt + Tools 定义就已经超过 7000 tokens，直接撑爆上下文窗口，导致模型拒绝生成。

### 解决方案

1. 在 LM Studio 右侧 **Configuration** 面板
2. 将 **Context Length** 从 `4096` 调大至 `16384`（推荐）或 `32768`
3. **重新加载模型**（必须，否则新 Context Length 不生效）

**内存参考**（24GB MacBook）：
| Context Length | 适用场景 | 备注 |
|----------------|----------|------|
| 8K | 单文件代码生成 | 极致流畅 |
| 16K | 多轮对话 + 工具调用 | 推荐 |
| 32K | 跨文件重构 | 无风扇机型可能降频 |

---

## 🛠️ 快速开始

### 1. LM Studio 服务端配置
1. 下载并加载 Gemma 4 12B GGUF 模型（推荐 Q4_K_M 或 Q5_K_M 量化版）
2. 在右侧 Configuration 面板设置：
    *   **Context Length**: `16384` (或 `32768`)
    *   **Prompt Format**: `Custom` (导入本项目 `templates/` 下的修复版模板)
3. 启动本地服务器 (默认 `http://localhost:1234`)

### 2. OpenCode 客户端配置
将本项目 `config/opencode-config.json` 的内容复制到你的 OpenCode 配置文件中。

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "lmstudio": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "LMStudio",
      "options": {
        "baseURL": "http://localhost:1234/v1",
        "apiKey": "your_key_api",
        "timeout": 600000
      },
      "models": {
        "gemma-4-12b-agentic-fable5-composer2.5-v2-3.5x-tau2": {
          "name": "gemma-4-12b-agentic-fable5-composer2.5-v2-3.5x-tau2",
          "contextWindow": 16000,
          "tool_call": true
        }
      }
    }
  }
}
```

**关键参数说明：**
| 参数 | 值 | 作用 |
|------|-----|------|
| `timeout` | `600000` | 10分钟超时，防止本地慢速生成大代码块时断开连接 |
| `tool_call` | `true` | 开启 Agent 工具调用能力，OpenCode 才会注入 Tool JSON Schema |
| `contextWindow` | `16000` | 对应 LM Studio 的 16K Context Length 设置 |

> **注意**：`apiKey` 随便填，`models` 下的 key 必须与 LM Studio 加载的模型 ID 严格一致。


---

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！如果你在接入其他模型（如 Qwen 2.5, Llama 3）时遇到了类似的模板或配置问题，欢迎将你的修复方案补充到 `templates/` 目录中。

## 📄 License

MIT License.
