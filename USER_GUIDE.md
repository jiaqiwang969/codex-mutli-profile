# 模型提供方池（Model Provider Pool）使用说明

## 基线信息
- 补丁基于 Codex 源码 commit：`0f8bb4579bd7c0ea905df7124b3a42835159b023`

## 功能说明
- 在 `config.toml` 中新增 `model_provider_pool`，用于自动切换 provider。
- 触发切换：缺少 API Key、鉴权失败（401/403）、限流（429）。
- 其他可重试错误：当当前 provider 的 stream 重试预算耗尽后切换。
- 切换后会把 `model_provider` 写回当前 profile（如有）否则写回全局。

## 配置方式
支持全局与 profile 两种配置（profile 优先级更高）。

```toml
model_provider = "ppai"
model_provider_pool = ["ppai", "vector", "openai"]

[model_providers.ppai]
name = "PPAI"
base_url = "https://example-ppai.com/v1"
env_key = "PPAI_API_KEY"
wire_api = "responses"

[model_providers.vector]
name = "Vector"
base_url = "https://example-vector.com/v1"
env_key = "VECTOR_API_KEY"
wire_api = "responses"

[profiles.gemini]
model_provider = "gemini"
model_provider_pool = ["gemini", "openai"]
```

## 行为细节
- profile 的 `model_provider_pool` 会覆盖全局配置。
- 如果当前 provider 在池中，会按顺序尝试下一个，并回环到起点（同一 turn 内每个 provider 只尝试一次）。
- 如果当前 provider 不在池中，则从池的第一个开始尝试。
- 缺失 `env_key` 或无法读取 API Key 的 provider 会被跳过。
- 请确保同一个 model 名称在池内所有 provider 都可用。

## 第一性原理（逻辑解释）
核心目标是“尽量不中断会话”。因此将“账号”抽象为 provider，把多个账号按优先级排列成池。
发生明确失败信号（鉴权/限流/重试耗尽）时，自动迁移到池里的下一个账号。
为了避免来回抖动，同一 turn 内每个账号只尝试一次；只要成功，就继续使用该账号并写回配置，保证后续稳定。
profile 独立池是为了不同场景（codex/gemini/grok 等）能用不同账号组，互不干扰。

## 账号池示例（12 个账号）
下面示例展示 12 个账号（每个账号对应一个独立的 API Key）：

```toml
model_provider = "ppai-01"
model_provider_pool = [
  "ppai-01", "ppai-02", "vector-01", "vector-02",
  "openai-01", "openai-02", "grok-01", "grok-02",
  "gemini-01", "gemini-02", "claude-01", "deepseek-01"
]

[model_providers.ppai-01]
name = "PPAI 01"
base_url = "https://api.ppai.example/v1"
env_key = "PPAI_API_KEY_01"
wire_api = "responses"

[model_providers.ppai-02]
name = "PPAI 02"
base_url = "https://api.ppai.example/v1"
env_key = "PPAI_API_KEY_02"
wire_api = "responses"

[model_providers.vector-01]
name = "Vector 01"
base_url = "https://api.vector.example/v1"
env_key = "VECTOR_API_KEY_01"
wire_api = "responses"

[model_providers.vector-02]
name = "Vector 02"
base_url = "https://api.vector.example/v1"
env_key = "VECTOR_API_KEY_02"
wire_api = "responses"

[model_providers.openai-01]
name = "OpenAI 01"
base_url = "https://api.openai.example/v1"
env_key = "OPENAI_API_KEY_01"
wire_api = "responses"

[model_providers.openai-02]
name = "OpenAI 02"
base_url = "https://api.openai.example/v1"
env_key = "OPENAI_API_KEY_02"
wire_api = "responses"

[model_providers.grok-01]
name = "Grok 01"
base_url = "https://api.grok.example/v1"
env_key = "GROK_API_KEY_01"
wire_api = "responses"

[model_providers.grok-02]
name = "Grok 02"
base_url = "https://api.grok.example/v1"
env_key = "GROK_API_KEY_02"
wire_api = "responses"

[model_providers.gemini-01]
name = "Gemini 01"
base_url = "https://generativelanguage.googleapis.com/v1beta"
env_key = "GEMINI_API_KEY_01"
wire_api = "gemini"

[model_providers.gemini-02]
name = "Gemini 02"
base_url = "https://generativelanguage.googleapis.com/v1beta"
env_key = "GEMINI_API_KEY_02"
wire_api = "gemini"

[model_providers.claude-01]
name = "Claude 01"
base_url = "https://api.anthropic.example/v1"
env_key = "CLAUDE_API_KEY_01"
wire_api = "responses"

[model_providers.deepseek-01]
name = "DeepSeek 01"
base_url = "https://api.deepseek.example/v1"
env_key = "DEEPSEEK_API_KEY_01"
wire_api = "responses"
```

## 使用测试目录
1. 复制现有配置到测试目录，例如 `~/.codex-test/xxxx`。
2. 在 `~/.codex-test/xxxx/config.toml` 中加入 `model_provider_pool`。
3. 运行 Codex 时设置 `CODEX_HOME` 指向测试目录。
4. 触发一次失败，确认后台输出切换提示信息。
