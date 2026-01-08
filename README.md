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

## 行为说明
- profile 的 `model_provider_pool` 会覆盖全局配置。
- 触发切换：缺少 API Key、401/403/429、或当前 provider 重试预算耗尽。
- 回环规则：从当前 provider 的下一个开始，回环直到尝试完池内每个 provider（同一 turn 内每个 provider 只尝试一次）。
- 自动持久化：切换后会把 `model_provider` 写回当前 profile（如有）否则写回全局。
- 请确保同一个 model 名称在池内所有 provider 都可用。

## 测试建议
建议复制 `~/.codex/` 到测试目录（如 `~/.codex-test/xxxx`），在其中加入 `model_provider_pool`，运行时将 `CODEX_HOME` 指向该目录。
