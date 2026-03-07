# Barney

![Barney](./assets/barney-main.png)

Barney is an OpenClaw skill for live Hinge automation on iOS via Appium + WebDriverAgent.

It can:
- observe your manual in-app behavior briefly (warmup),
- switch into autonomous mode,
- triage Discover/Likes/Standouts/Chats by selected mode,
- generate or rewrite messages with OpenAI,
- keep structured activity logs in `hinge-data/`.

## What It Does

- Real-time iOS control of Hinge (`activate`, `snapshot`, `go-tab`, `open-thread`, scroll, like/reply/rose actions by mode policy).
- Agent modes:
  - `like_only`
  - `full_access`
  - `likes_with_comments_only`
- Warmup learning before takeover:
  - default: `90s`
  - learns from inferred actions and typed style
  - persists observation summary + hints into profile preferences
- Activity and state tracking:
  - `agent-state.json`
  - `activity-log.json`
  - `analysis-latest.json`
  - screenshots/artifacts in `activity/<date>/images`

## How It Works

1. Daemon creates/attaches Appium session to iPhone.
2. Optional observe phase runs before autonomous actions.
3. Planner/evaluator decides per-profile or per-thread action.
4. Policy guardrails enforce the selected mode.
5. Results + telemetry are written to `hinge-data/`.

Core scripts:
- `scripts/hinge-agent-daemon.js`: long-running orchestrator
- `scripts/discover-autopilot.js`: per-cycle planning/execution
- `scripts/hinge-ai.js`: OpenAI + style passes + message shaping
- `scripts/hinge-ios.js`: Hinge-specific iOS actions
- `scripts/appium-ios.js`: Appium session helpers
- `scripts/onboarding.js`: workspace init/validation/config helpers

## Quick Start

### 1. Prereqs

- macOS + Xcode
- Node.js 18+ (required for native `fetch` and `node:readline/promises`)
- iPhone connected and trusted
- Appium runtime workspace (`~/.openclaw/workspaces/hinge-automation`)
- Hinge installed and logged in on phone
- OpenAI API key

### 2. Initialize local skill data

```bash
node scripts/onboarding.js --init --dir hinge-data
node scripts/onboarding.js --validate --dir hinge-data
```

### 3. Configure basics

In `hinge-data/profile-preferences.json`:
- set your user preferences/goals/tone
- set iOS values (`ios.deviceName`, `ios.udid`, `ios.bundleId`)
- set mode defaults under `automation`

### 4. Launch daemon

```bash
node scripts/hinge-agent-daemon.js --launch --dir hinge-data
```

Optional:
- `--agent-mode like_only|full_access|likes_with_comments_only`
- `--observe-seconds 60`
- `--skip-observe`

### 5. Monitor live

```bash
node scripts/hinge-agent-daemon.js --status --dir hinge-data
tail -f hinge-data/agent.log
tail -f hinge-data/appium.log
```

## Keys and Secrets

Required:
- `OPENAI_API_KEY`

Optional:
- custom model IDs in config (for fine-tuned rewrite flows)

Notes:
- Fine-tuned model access is scoped by OpenAI org/project permissions.
- Runtime now defaults to base models only (`--use-base-model-only true`).
- Do not commit `hinge-data/` or secret-bearing configs.

## Modes

- `like_only`: likes + roses only, no comments/replies.
- `full_access`: likes/comments/roses/replies.
- `likes_with_comments_only`: likes (with or without comments), no replies, no roses.

Mode is selected at launch prompt or via `--agent-mode`.

## Data Paths

Default runtime directory:
- `hinge-data/` (or custom `--dir`)

Important files:
- `profile-preferences.json`
- `agent-state.json`
- `activity-log.json`
- `user-observation.json`
- `agent.log`
- `appium.log`

## Packaging for GitHub / ClawHub

Build clean archives:

```bash
bash scripts/package-skill.sh
```

Outputs:
- `barney-github-ready.zip`
- `barney-clawhub-ready.zip`

Excluded by packaging:
- `hinge-data/`
- `node_modules/`
- cache/log/archive artifacts

## Troubleshooting

- `Unknown device or simulator UDID`: reconnect phone, trust prompt, verify UDID.
- WDA start loops / socket hang up: ensure Xcode signing + WebDriverAgent works on device.
- Stuck state: check `agent-state.json` + `appium.log` first.
- No OpenAI output: verify `OPENAI_API_KEY` and model access permissions.

## Safety and Use

Use with explicit user consent and account ownership. Follow platform terms and local law. Keep automation respectful and avoid spammy behavior.
