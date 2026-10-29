# Hermes Jev Routing

<div align="center">
  <img src="docs/hero.png" width="100%" alt="Jev Routing" />
</div>

Per-turn model routing for Hermes Agent powered by TypeSafe Jev via OpenRouter.

Originally authored by Apostol Apostolov. Refactored using **REA (Reverse Engineer Anything)** and tidied up with **Antigravity** by Philip ([@psthi](https://github.com/psthi)).

---

## Overview

`hermes-jev-routing` intercepts prompts before model execution. Jev evaluates the user prompt, short history context, estimated token footprint, and spend snapshot across four typed questions:
- **Task Kind:** Classification of intent (`plan`, `implement`, `debug`, `refactor`, `explain`, `chat`, etc.)
- **Complexity:** Difficulty rating from trivial to architectural
- **Capability Deserved:** Model tier justification
- **Deep Reasoning Need:** Likelihood that extended chain-of-thought is required

The router maps these metrics to your custom route table tiers (`quick`, `standard`, `high`, `premium`) and dynamically rewrites the outbound request to the optimal model without cross-provider overhead.

### OpenRouter Native Decision API
Unlike upstream implementations relying on direct TypeSafe endpoints, this edition routes directly through **OpenRouter's Alpha Decisions API** (`https://openrouter.ai/api/alpha/decisions`), seamlessly normalizing model identifiers to OpenRouter's `~typesafe/jev-latest` namespace. No standalone TypeSafe account or direct billing credential is required—everything routes under your existing `OPENROUTER_API_KEY`.

---

## Installation

```bash
hermes plugins install psthi/hermes-jev-routing
```

Or clone directly into your Hermes plugins directory:

```bash
git clone https://github.com/psthi/hermes-jev-routing.git ~/.hermes/plugins/hermes-jev-routing
```

Requires Hermes Agent `0.21.4` or newer and an `OPENROUTER_API_KEY` in `~/.hermes/.env`.

---

## Configuration

### 1. Enable in `~/.hermes/config.yaml`

Add the plugin to `plugins.enabled` and configure its entry in `~/.hermes/config.yaml`:

```yaml
plugins:
  enabled:
    - hermes-jev-routing
  entries:
    hermes-jev-routing:
      settings:
        mode: auto           # Options: off, shadow, notify, auto, confirm
        config_path: ~/.hermes/jev-routes.json
```

| Mode | Behavior |
| --- | --- |
| `shadow` | (Default) Classifies tasks and logs recommendations without rewriting requests. |
| `auto` | Automatically rewrites the provider request to the selected model tier. |
| `notify` | Logs and displays the selected tier under the composer. |
| `confirm` | Prompts for confirmation before stepping up to higher tiers. |
| `off` | Disables routing entirely. |

---

### 2. Route Table (`~/.hermes/jev-routes.json`)

Create `~/.hermes/jev-routes.json` to define your provider and model tiers:

```json
{
  "enabled": true,
  "confidenceThreshold": 0.34,
  "minPromptChars": 12,
  "routes": {
    "quick": [
      { "provider": "openrouter", "model": "deepseek/deepseek-v4-flash" }
    ],
    "standard": [
      { "provider": "openrouter", "model": "google/gemini-3.8-flash" }
    ],
    "high": [
      { "provider": "openrouter", "model": "anthropic/claude-opus-5.5" }
    ],
    "premium": [
      { "provider": "openrouter", "model": "anthropic/claude-fable-5.1" }
    ],
    "xpremium": []
  },
  "kindMinimumTier": {
    "plan": "high",
    "review": "high",
    "implement": "standard",
    "chat": "quick"
  },
  "confirm": {
    "tiers": ["premium", "xpremium"],
    "onTimeout": "reject"
  },
  "budget": {
    "dailyUsd": 10.0,
    "monthlyUsd": 150.0,
    "softRatio": 0.7,
    "hardRatio": 0.9
  }
}
```

---

## Slash Commands & Interactive Tools

Inside Hermes CLI or Gateway chats:

| Command | Action |
| --- | --- |
| `/jev-routing` | View active mode, spend metrics, route table chains, and last decision. |
| `/jev-routing why` | Inspect full Jev evaluation for the last prompt. |
| `/jev-routing route <text>` | Dry-run classification on arbitrary input without switching models. |
| `/jev-routing revert` | Revert to the model active prior to the last routing switch. |
| `/jev-routing mode <name>` | Switch routing mode (`auto`, `shadow`, `confirm`, `off`) for the current session. |
| `/jev-routing on` / `off` | Quickly toggle routing for the session. |
| `jev_route` tool | Model tool enabling agents to query tier requirements for delegated tasks. |

---

## Spend Tracking

Every turn logs token and monetary usage into `<config-stem>-state.json` beside your route configuration file. Spend pressures feed dynamically into routing decisions to ensure budget caps are respected.

---

## License

[MIT](LICENSE)
