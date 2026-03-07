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

### 1. Mac and Xcode setup

Install Xcode from the App Store if you haven't already. After installing, open it once so it finishes component setup, then install the command line tools:

```bash
xcode-select --install
```

Make sure Xcode is up to date. WebDriverAgent requires a recent Xcode and a valid Apple developer account (a free account works).

### 2. Prepare your iPhone

**Enable Developer Mode on the device:**

1. Go to **Settings → Privacy & Security → Developer Mode** and turn it on.
2. The phone will reboot and ask you to confirm — tap **Turn On**.

> Developer Mode is required on iOS 16 and later. Without it, Appium cannot control the device.

**Connect and trust:**

1. Plug the iPhone into your Mac with a USB cable.
2. On the phone, tap **Trust** when the "Trust This Computer?" prompt appears.
3. Enter your passcode if asked.

**Verify the Mac sees it:**

```bash
xcrun xctrace list devices
```

You should see your iPhone listed with a UDID that looks like `00008120-001234AB3C04401E`. Copy this — you will need it in step 6.

### 3. Install Node.js 18+

Node.js 18 or later is required (for native `fetch` and `node:readline/promises`). Check your version:

```bash
node --version
```

If it is below 18, install via [nodejs.org](https://nodejs.org) or with a version manager:

```bash
# with nvm
nvm install 20
nvm use 20
```

### 4. Set up the Appium workspace

Barney expects an Appium runtime workspace at `~/.openclaw/workspaces/hinge-automation`. Create it and install Appium there:

```bash
mkdir -p ~/.openclaw/workspaces/hinge-automation
cd ~/.openclaw/workspaces/hinge-automation
npm init -y
npm install appium
npx appium driver install xcuitest
npx appium driver list
```

The last command should show `xcuitest` as installed. Run the doctor to catch any missing dependencies:

```bash
npx appium driver doctor xcuitest
```

Common fixes it may suggest: install `libimobiledevice` and `ios-deploy` via Homebrew.

```bash
brew install libimobiledevice ios-deploy
```

### 5. Sign WebDriverAgent in Xcode

Appium uses WebDriverAgent (WDA) to talk to your phone. It needs to be signed with your Apple ID before it will run on a real device.

1. Open Xcode.
2. In the menu bar: **Xcode → Open Developer Tool → Simulator** — just to confirm Xcode is active.
3. Find the WDA project. It lives inside the xcuitest driver you just installed:

```bash
find ~/.npm -name "WebDriverAgent.xcodeproj" 2>/dev/null | head -1
# or
find /usr/local -name "WebDriverAgent.xcodeproj" 2>/dev/null | head -1
```

4. Open `WebDriverAgent.xcodeproj` in Xcode.
5. Select the **WebDriverAgentLib** target → **Signing & Capabilities** tab.
6. Under Team, sign in with your Apple ID (free account is fine). Change the Bundle Identifier to something unique, e.g. `com.yourname.wda`.
7. Repeat for the **WebDriverAgentRunner** target.
8. Connect your iPhone, select it as the build target, and do **Product → Build** (`⌘B`). It should build without errors.

> If you see a "device is not registered" error, go to **Settings → General → VPN & Device Management** on your phone and trust the developer certificate.

### 6. Find your device values

You need three values for the config:

```bash
# UDID (from step 2, or re-run):
xcrun xctrace list devices

# iOS version:
# Check Settings → General → About → iOS Version on the phone

# Hinge bundle ID (usually):
# co.hinge.mobile.ios
```

### 7. Set your OpenAI API key

```bash
export OPENAI_API_KEY=sk-...
```

Add it to your shell profile (`~/.zshrc` or `~/.bash_profile`) to persist it across sessions.

### 8. Initialize local skill data

Run this from the `barney/` skill directory:

```bash
node scripts/onboarding.js --init --dir hinge-data
```

This creates `hinge-data/` with a starter `profile-preferences.json`.

### 9. Configure your preferences

Open `hinge-data/profile-preferences.json` and fill in:

```json
{
  "user": {
    "goals": "looking for something serious",
    "tone": "playful",
    "dealbreakers": ["smoker", "no ambition"]
  },
  "ios": {
    "deviceName": "iPhone",
    "udid": "YOUR-UDID-HERE",
    "bundleId": "co.hinge.mobile.ios",
    "platformVersion": "18.0"
  },
  "automation": {
    "agentMode": "full_access"
  }
}
```

Then validate:

```bash
node scripts/onboarding.js --validate --dir hinge-data
```

It will tell you if anything required is missing.

### 10. Start Appium

Open a separate terminal, go to the Appium workspace, and start the server:

```bash
cd ~/.openclaw/workspaces/hinge-automation
npx appium --base-path / -p 4725 --log-level warn
```

Leave this terminal running. You should see `Appium REST http interface listener started`.

### 11. Open Hinge on your phone

Unlock the phone and open the Hinge app manually. Make sure you are logged in and the app is in the foreground.

### 12. Launch the agent

Back in the `barney/` directory:

```bash
node scripts/hinge-agent-daemon.js --launch --dir hinge-data
```

You will be prompted to choose a mode unless you pass `--agent-mode`:

```bash
# Examples:
node scripts/hinge-agent-daemon.js --launch --dir hinge-data --agent-mode like_only
node scripts/hinge-agent-daemon.js --launch --dir hinge-data --agent-mode full_access --observe-seconds 60
node scripts/hinge-agent-daemon.js --launch --dir hinge-data --skip-observe
```

The daemon will:
1. Connect to your running Appium server.
2. Attach to the Hinge session on your phone.
3. Observe your manual usage for the warmup period (default 90s).
4. Take over and begin the selected triage mode.

### 13. Monitor live

```bash
# Check current agent state:
node scripts/hinge-agent-daemon.js --status --dir hinge-data

# Follow live logs:
tail -f hinge-data/agent.log
tail -f hinge-data/appium.log
```

### 14. Stop the agent

```bash
node scripts/hinge-agent-daemon.js --stop --dir hinge-data
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
