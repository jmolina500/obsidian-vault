# Using Single Provider Configuration

Created: 2025-04-20

## Goal
Configure Hermes to use only one specific LLM provider, preventing automatic fallback to other providers.

## Steps

### 1. Set Your Preferred Provider

```bash
hermes config set model.provider <provider_name>
```

**Examples:**
```bash
hermes config set model.provider openrouter
hermes config set model.provider kimi-coding
hermes config set model.provider anthropic
```

### 2. Remove Other API Keys

Edit your `.env` file and remove or comment out API keys for providers you don't want to use:

```bash
hermes config env-path   # Shows path to .env file
# Edit the file and keep only your chosen provider's API key
```

### 3. Verify Configuration

```bash
hermes config      # View current config
hermes doctor      # Check everything is configured correctly
```

## Provider Reference

Common provider names for `model.provider`:

| Provider | Config Value | Environment Variable |
|----------|--------------|---------------------|
| OpenRouter | `openrouter` | `OPENROUTER_API_KEY` |
| Moonshot/Kimi | `kimi-coding` | `KIMI_API_KEY` |
| Anthropic | `anthropic` | `ANTHROPIC_API_KEY` |
| OpenAI | `openai` | `OPENAI_API_KEY` |
| DeepSeek | `deepseek` | `DEEPSEEK_API_KEY` |
| Google Gemini | `google` | `GOOGLE_API_KEY` |

## How to Check Current Provider

The provider is shown at the start of each conversation:

```
Conversation started: Monday, April 20, 2026 08:27 AM
Model: kimi-k2.5
Provider: kimi-coding
```

## Related Commands

```bash
hermes model              # Interactive model/provider picker
hermes setup model        # Setup wizard for model configuration
hermes auth list          # List configured credentials
```

## Notes

- The provider field displays the configuration name (e.g., `kimi-coding`, `openrouter`)
- The actual service company may differ (e.g., `kimi-coding` → Moonshot AI)
- Removing API keys from `.env` prevents accidental fallback if the primary provider fails
