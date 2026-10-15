<a id="readme-top"></a>

<div align="center">
  <h1>pi-fast-mode</h1>
  <p>Provider-aware Fast mode for Pi, with per-child control for subagents and workflow steps.</p>
  <p>
    <a href="#features">Features</a> ·
    <a href="#getting-started">Install</a> ·
    <a href="#usage">Usage</a> ·
    <a href="#configuration">Configuration</a>
  </p>
</div>

[![MIT License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## Table of Contents

- [About](#about)
- [Features](#features)
- [Getting Started](#getting-started)
- [Usage](#usage)
  - [Control child agents](#control-child-agents)
  - [pi-subagents](#pi-subagents)
  - [pi-dynamic-workflows](#pi-dynamic-workflows)
- [Supported providers](#supported-providers)
- [Configuration](#configuration)
- [Development](#development)
- [License](#license)
- [Acknowledgments](#acknowledgments)

## About

`pi-fast-mode` is a standalone Pi extension based on Diego Petrucci's `fast` extension. It provides the `/fast` and `/ultrafast` commands, and adds launch-time Fast-mode preferences for child agents.

This is useful for extensive agent orchestration. Enable Fast mode only for the subagents or workflow steps that need it, without turning it on for the parent session or every agent you start. The launch preference applies to that child session only.

## Features

- Select provider-specific Fast request settings for supported OpenAI, OpenAI Codex, and Anthropic models.
- Set a different Fast-mode preference for each child through `pi:agent-launch-context` metadata.
- Keep launch metadata out of the model prompt and leave the parent session unchanged.
- Fail closed for unknown launch preferences and unsupported Ultrafast requests.
- Preserve any existing provider `speed` or `service_tier` field.

## Getting Started

### Requirements

- Pi
- Node.js 22.19.0 or newer

### Install

```sh
pi install git:github.com/codesoda/pi-fast-mode
```

Then reload Pi:

```text
/reload
```

## Usage

### Session commands

```text
/fast
/ultrafast
/fast ultrafast
```

`/fast` toggles ordinary Fast mode for the current session. It follows model changes and uses the request setting for the selected provider. Fast mode is off by default.

`/ultrafast` and `/fast ultrafast` toggle the session-only Ultrafast mode. Ultrafast is never enabled by configuration. If the selected model or endpoint is not eligible, Ultrafast turns off and does not fall back to ordinary Fast.

### Control child agents

An orchestrator can add a `pi:agent-launch-context` custom entry to the child session. The entry stores this data payload:

```json
{
  "version": 1,
  "agentType": "reviewer",
  "metadata": { "fast": true }
}
```

The extension reads `metadata.fast` from the child session's latest `pi:agent-launch-context` entry. It accepts:

| Value | Child behavior |
|---|---|
| `true`, `"true"`, or `"fast"` | Enable ordinary Fast mode. |
| `false`, `"false"`, or `"off"` | Keep Fast mode off, even if configuration enables it. |
| `"ultrafast"` | Request Ultrafast, subject to its stricter eligibility checks. |
| Missing | Keep normal configuration and session behavior. |
| Any other value | Fail closed and keep Fast mode off. |

If the child starts before a model is selected, an Ultrafast request waits for its first model selection. It activates only if that model is eligible. Launch preferences do not change the parent session.

### pi-subagents

Add `fast: true` to a custom agent's frontmatter to enable ordinary Fast mode for that agent's child session:

```yaml
---
fast: true
---

Review the requested changes and report only actionable findings.
```

Leave the field out for agents that should use normal settings. Use `fast: "off"` to explicitly disable Fast mode for a selected agent.

### pi-dynamic-workflows

Define a fast-enabled agent, then select it only for workflow steps that need it:

```yaml
---
name: fast-reviewer
fast: true
---

Review the requested changes and report only actionable findings.
```

```ts
await agent("Review this patch for regressions.", { agentType: "fast-reviewer" });
```

Only steps that select `fast-reviewer` start with Fast mode. Other steps can use an agent definition without `fast: true` or the workflow's normal agent settings. This lets a workflow choose Fast mode step by step rather than enabling it for every agent in the run.

## Supported providers

### Direct OpenAI API

Fast mode requires provider `openai`, API `openai-responses` or `openai-completions`, and one of these models:

- `gpt-6.1-sol`
- `gpt-6-astra`
- `gpt-6-sol`
- `gpt-6-luna`

The extension requests `service_tier: "fast"` when the request does not already set a tier. It supports API-key auth and Pi's OpenAI ChatGPT OAuth. The OAuth request shape has not been live-verified.

### OpenAI Codex

Fast mode requires provider `openai-codex`, API `openai-codex-responses`, ChatGPT OAuth, and one of these models:

- `gpt-5.5`
- `gpt-5.6-sol`
- `gpt-5.6-terra`
- `gpt-5.6-luna`
- `gpt-6-astra`
- `gpt-6.1-sol`
- `gpt-6-sol`
- `gpt-6-luna`

The extension requests `service_tier: "priority"`. It does not apply this mode to API-key auth.

### Anthropic Claude

Fast mode requires provider `anthropic`, API `anthropic-messages`, and model `claude-opus-4-8`, `claude-opus-5`, or `claude-opus-5-5`. It requests `speed: "fast"` and adds the required Fast-mode beta header. API-key and Claude Code OAuth access work when the account has access to the research preview.

### Ultrafast

Ultrafast has narrower support than ordinary Fast. It requires all of the following:

- Provider `openai`
- API `openai-responses`
- Model `gpt-6-astra`
- Direct OpenAI API-key auth
- The canonical `https://api.openai.com/v1` endpoint (one trailing slash is accepted)

Eligible requests receive `service_tier: "ultrafast"` only when they do not already set a tier. Ultrafast does not support Codex OAuth, OpenAI OAuth, Chat Completions, proxy or regional endpoints, or Anthropic. It never falls back to ordinary Fast.

**Cost warning:** OpenAI lists Ultrafast at 6× Standard pricing. Pi's current cost display does not include that premium. The extension requests a tier; it cannot confirm that the provider accepted or applied it. See the [Ultrafast API guide](https://developers.openai.com/api/docs/guides/ultrafast-mode) and [API pricing](https://developers.openai.com/api/docs/pricing) before use.

Fast availability and pricing depend on the provider, account, plan, and rollout. Local eligibility checks do not certify live provider behavior. Pi may not report the effective tier in usage data, and this extension does not rewrite Pi's cost estimates.

## Configuration

Optional global config:

```text
<pi-config-dir>/agent/extensions/fast.json
```

Optional project config, from the nearest ancestor of the working directory:

```text
<project>/<pi-config-dir>/fast.json
```

`<pi-config-dir>` is Pi's runtime config directory (`.pi` by default). Project config is read only when Pi trusts the project. Copy `extensions/fast/fast.example.json` as a starting point:

```json
{
  "enabled": false,
  "showStatus": true
}
```

- `enabled`: Default ordinary Fast-mode state when there is no session override.
- `showStatus`: Show a compact status when Fast mode is active for the selected model.

Configuration cannot enable Ultrafast. The extension reads `fast.json`; it does not read the deprecated `openai-fast.json` or `claude-fast.json` files.

## Development

```sh
npm install
npm test
npm run typecheck
```

The tests use mocked models and requests. They do not make live or paid provider calls.

## License

Distributed under the MIT License. See [LICENSE](LICENSE).

## Acknowledgments

This extension started from Diego Petrucci's [original `fast` extension](https://github.com/diegopetrucci/pi-extensions/tree/main/extensions/fast). Its per-agent launch-mode support was first proposed in [Codesoda's unmerged PR #159](https://github.com/diegopetrucci/pi-extensions/pull/159); this repository carries that work forward as a standalone package.

<p align="right"><a href="#readme-top">Back to top</a></p>
```