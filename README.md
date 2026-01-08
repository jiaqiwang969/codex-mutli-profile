# codex-mutli-profile

这是对 Codex 的补丁，增加“账号池/模型提供方池”的自动切换能力，支持按 profile 单独配置，并在失败时回环切换。

## 基线信息
- 补丁基于 Codex 源码 commit：`0f8bb4579bd7c0ea905df7124b3a42835159b023`

## 文件说明
- `codex-provider-pool.patch`：补丁文件
- `USER_GUIDE.md`：详细说明（中文）

## 应用补丁
在 Codex 源码根目录执行：

```bash
git apply /path/to/codex-provider-pool.patch
```

## 配置示例（config.toml）
全局配置：

```toml
model_provider = "openai-proxy"
model_provider_pool = ["openai-proxy", "gemini"]

[model_providers.openai-proxy]
name = "Codex OpenAI Proxy"
base_url = "https://api.vectorengine.ai/v1"
wire_api = "responses"
requires_openai_auth = true

[model_providers.gemini]
name = "Codex Gemini Proxy"
base_url = "https://generativelanguage.googleapis.com/v1beta"
wire_api = "gemini"
requires_openai_auth = true
auth_json_key = "GEMINI_API_KEY"
```

按 profile 单独配置：

```toml
[profiles.codex]
model_provider = "openai-proxy"
model_provider_pool = ["openai-proxy", "gemini"]

[profiles.gemini]
model_provider = "gemini"
model_provider_pool = ["gemini", "openai-proxy"]
```

## 第一性原理（逻辑解释）
目标是“在不打断会话的情况下最大化可用性”。因此把每个账号视为一个可替换的 provider，按优先级组成池。
当出现确定的失败信号（如鉴权/限流/重试耗尽），就把请求迁移到池里的下一个账号。
为避免来回抖动，同一 turn 内每个账号只尝试一次；如果成功，就继续使用当前账号并把选择写回配置，保证后续 turn 稳定。
profile 独立池是为了不同场景（如 codex/gemini/grok）使用不同账号组，互不干扰。

## 行为说明
- profile 的 `model_provider_pool` 会覆盖全局配置。
- 触发切换：缺少 API Key、401/403/429、或当前 provider 重试预算耗尽。
- 回环规则：从当前 provider 的下一个开始，回环直到尝试完池内每个 provider（同一 turn 内每个 provider 只尝试一次）。
- 自动持久化：切换后会把 `model_provider` 写回当前 profile（如有）否则写回全局。
- 请确保同一个 model 名称在池内所有 provider 都可用。

## 账号池示例（12 个账号）
下面示例展示 12 个账号（每个账号用不同的 `env_key` 表示不同 API Key）。你可以按同样模式扩展：

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

## 测试建议
建议复制 `~/.codex/` 到测试目录（如 `~/.codex-test/xxxx`），在其中加入 `model_provider_pool`，运行时将 `CODEX_HOME` 指向该目录。
