# codex-mutli-profile

这是一个 Codex 补丁：**Provider 在会话开始就确定**，之后只在该 Provider 的“账号池”内自动轮询切换（每个账号是 `base_url + env_key` 绑定对）。切换发生在会话内，不需要手动改 `config.toml`，也不需要退出重进。

## 基线信息
- 补丁基于 Codex 源码 commit：`0f8bb4579bd7c0ea905df7124b3a42835159b023`

## 文件说明
- `codex-account-pool.patch`：补丁文件

## 应用补丁
在 Codex 源码根目录执行：

```bash
git apply /path/to/codex-account-pool.patch
```

## 第一性原理（逻辑解释）
目标不是“把多个 Provider 混在一起”，而是 **在用户已经选定 Provider 的前提下，提高可用性**。
换句话说：
- Provider（如 OpenAI/Gemini/Grok）在会话一开始就确定下来。
- 之后只在这个 Provider 的账号池里轮询，避免跨 Provider 的不确定性。
- 每个账号是一对 `(base_url, env_key)`，失败就切下一个，不用手动改配置。

这样做的好处：
- **不中断**：切换发生在当前会话内，不需要退出/重进。
- **稳定**：同一 turn 里每个账号只尝试一次，避免抖动。
- **省心**：自动写回 `config.toml`，下次直接用上次成功的账号。

## 核心概念
- **Provider**：会话开始选择的模型提供方（例如 openai / gemini / grok）。
- **账号（Account）**：`base_url + env_key` 绑定对。
- **账号池（account_pool）**：Provider 内的账号列表，**有顺序**，顺序就是优先级。

## 轮询与回环（人话 + 插图）
人话解释：
- 如果当前账号报错（如 401/403/429、重试耗尽、或缺失 API Key），就切到下一个账号。
- 到列表末尾就回到列表开头继续尝试，这就是“回环”。
- 同一 turn 内每个账号只试一次；都不行就退出本次 turn。

插图示意（同一 Provider 内回环）：

```
Provider = openai
账号池顺序：A1 -> A2 -> A3 -> A4 -> A5 -> A6 -> A7 -> A8 -> A9 -> A10 -> A11 -> A12 -> (回到 A1)
当前账号：A4

本次 turn 失败链路：
A4 失败 -> A5 -> A6 -> A7 -> A8 -> A9 -> A10 -> A11 -> A12 -> A1 -> A2 -> A3 -> (本轮结束)

下一次 turn：从“上次成功的账号”继续工作
```

## 配置方式（profile 对应 Provider，Provider 内配置账号池）
你只需要在 `~/.codex/config.toml` 里配置：
1. profile 选择 provider；
2. provider 设置 `account_pool`。

示例（多个 profile 各自使用自己的 provider）：

```toml
[profiles.codex]
model_provider = "openai"

[profiles.gemini]
model_provider = "gemini"

[profiles.grok]
model_provider = "grok"

[model_providers.openai]
name = "OpenAI"
wire_api = "responses"
account_pool = [
  { base_url = "https://api.ppai.example/v1", env_key = "OPENAI_API_KEY_01" },
  { base_url = "https://api.vector.example/v1", env_key = "OPENAI_API_KEY_02" }
]

[model_providers.gemini]
name = "Gemini"
wire_api = "chat"
account_pool = [
  { base_url = "https://generativelanguage.googleapis.com/v1beta", env_key = "GEMINI_API_KEY_01" }
]

[model_providers.grok]
name = "Grok"
wire_api = "responses"
account_pool = [
  { base_url = "https://api.grok.example/v1", env_key = "GROK_API_KEY_01" }
]
```

> 如果你希望 “同一个 Provider 类型，但不同 profile 使用不同账号池”，可以定义多个 provider：
> 例如 `openai-work` 和 `openai-personal`，再让 profile 分别指向它们。

## 账号池大样例（>=10 个账号）
下面示例展示一个 Provider（openai）里 **12 个账号**，满足“账号池”场景：

```toml
[model_providers.openai]
name = "OpenAI"
wire_api = "responses"
account_pool = [
  { base_url = "https://api.ppai.example/v1", env_key = "OPENAI_API_KEY_01" },
  { base_url = "https://api.ppai.example/v1", env_key = "OPENAI_API_KEY_02" },
  { base_url = "https://api.vector.example/v1", env_key = "OPENAI_API_KEY_03" },
  { base_url = "https://api.vector.example/v1", env_key = "OPENAI_API_KEY_04" },
  { base_url = "https://api.openai.example/v1", env_key = "OPENAI_API_KEY_05" },
  { base_url = "https://api.openai.example/v1", env_key = "OPENAI_API_KEY_06" },
  { base_url = "https://api.proxy-a.example/v1", env_key = "OPENAI_API_KEY_07" },
  { base_url = "https://api.proxy-b.example/v1", env_key = "OPENAI_API_KEY_08" },
  { base_url = "https://api.proxy-c.example/v1", env_key = "OPENAI_API_KEY_09" },
  { base_url = "https://api.proxy-d.example/v1", env_key = "OPENAI_API_KEY_10" },
  { base_url = "https://api.proxy-e.example/v1", env_key = "OPENAI_API_KEY_11" },
  { base_url = "https://api.proxy-f.example/v1", env_key = "OPENAI_API_KEY_12" }
]

# 当前使用账号（Codex 会自动写回这两项）
base_url = "https://api.ppai.example/v1"
env_key = "OPENAI_API_KEY_01"
```

说明：
- `account_pool` 的顺序就是优先级。
- 每个账号必须同时有 `base_url` 和 `env_key`，缺一会被跳过。
- 你可以把 `base_url` 指向不同代理，只要它们遵循同一 Provider 的协议即可。

## 触发切换的条件
- 缺少 API Key（环境变量未设置）。
- HTTP 401 / 403 / 429。
- 该 Provider 的流式重试次数耗尽。

## 自动持久化与不中断
- 切换发生在会话内，不需要手动命令，不需要重启。
- Codex 会把选中的账号写回 `model_providers.<provider_id>.base_url` 与 `.env_key`。

## 测试建议（可选）
为了不影响日常配置，可以复制 `~/.codex/` 到测试目录，例如：
`~/.codex-test/xxxx`，然后把 `CODEX_HOME` 指向这个目录测试。
