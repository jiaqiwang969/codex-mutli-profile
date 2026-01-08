# Model Provider Pool Guide

## What it does
- Adds a `model_provider_pool` list to `config.toml` for automatic provider failover.
- On missing API keys, auth errors (401/403), or rate limit errors (429), Codex switches immediately.
- On other retryable errors, Codex switches after the current provider exhausts its stream retry budget.
- The selected provider is persisted to `model_provider` (active profile if set, otherwise top level).

## Configure
Add providers (with distinct IDs) and the pool ordering. You can set the pool
globally or per profile.

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

## Behavior details
- Profile pools override the top-level pool.
- If the current provider is in the pool, Codex tries the next entries in order and wraps back to the start, attempting each entry once per turn.
- If the current provider is not in the pool, Codex starts from the first entry in the pool.
- Pool entries that are missing `env_key` values are skipped.
- Ensure the configured model name is valid for every provider in the pool.

## Testing with ~/.codex-test/xxxx
1. Copy your existing config: `cp -a ~/.codex/. ~/.codex-test/xxxx/`
2. Edit `~/.codex-test/xxxx/config.toml` to add `model_provider_pool`.
3. Run Codex with `CODEX_HOME=~/.codex-test/xxxx`.
4. Trigger a failure on the active provider and confirm the background switch message.
