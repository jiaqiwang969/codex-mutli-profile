# codex-mutli-profile

这是一个 Codex 补丁：在用户选定 Provider 之后，自动在该 Provider 的账号池内轮询切换（每个账号是 base_url + env_key 绑定对）。切换发生在会话内，不需要手动改 config.toml，也不需要退出重进。

## 基线信息
- 补丁基于 Codex 源码 commit：`0f8bb4579bd7c0ea905df7124b3a42835159b023`

## 文件说明
- `codex-account-pool.patch`：补丁文件
- `USER_GUIDE.md`：详细说明（中文）

## 应用补丁
在 Codex 源码根目录执行：

```bash
git apply /path/to/codex-account-pool.patch
```

## 快速配置示例（config.toml）

```toml
[profiles.codex]
model_provider = "openai"

[profiles.gemini]
model_provider = "gemini"

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
```

更多细节、轮询逻辑说明、10+ 账号示例请看 `USER_GUIDE.md`。
