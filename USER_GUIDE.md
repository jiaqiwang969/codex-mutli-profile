# 模型提供方池（Model Provider Pool）使用说明

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

## 使用测试目录
1. 复制现有配置到测试目录，例如 `~/.codex-test/xxxx`。
2. 在 `~/.codex-test/xxxx/config.toml` 中加入 `model_provider_pool`。
3. 运行 Codex 时设置 `CODEX_HOME` 指向测试目录。
4. 触发一次失败，确认后台输出切换提示信息。
