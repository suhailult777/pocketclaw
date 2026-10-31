# PocketClaw

**My B.Tech major project.** PocketClaw is an Android AI chat app built with Expo and React Native. It runs in two ways: in the **cloud**, through NVIDIA NIM models, or **fully offline** with a GGUF model running on the phone via `llama.rn`. In agent mode the model can search the web, read pages, and (optionally) run a small allow-listed set of commands on a paired PC.

## Features

- **Two inference paths, one chat UI**
  - *Cloud (nvidia-proxy):* the app streams tokens from a small Express proxy that forwards OpenAI-style `/v1/chat/completions` calls to NVIDIA NIM. If the requested model fails, the proxy falls back to a quality or capacity model and reports which one answered (`x-model-selected`, `x-model-fallback-used`).
  - *Offline (native):* loads a `.gguf` file picked from the device and runs it on-device through `llama.rn`. No network needed.
- **Tool calling (agent mode):** `web_search` and `fetch_page`, plus an optional `zeroclaw_webhook` that forwards command requests to a paired gateway on your PC. Commands are checked against an allow-list (`server/openclawPolicy.js`: read-only things like `whoami`, `dir`, `Get-Content`) and have replay protection.
- **Streaming UX:** throttled token rendering, skeleton bubbles, and persisted settings (provider, host/port, model, agent mode).
- **Tests:** Vitest unit tests for settings persistence, the tool executor and the command policy, plus a proxy integration test.

## Architecture

```
ChatScreen ──▶ providerRouter.ts ──┬─▶ nvidiaProxyClient.ts ──▶ server/index.js (Express) ──▶ NVIDIA NIM
                                   │                               └─ tools: web_search, fetch_page, zeroclaw_webhook
                                   └─▶ nativeClient.ts ──▶ llama.rn (on-device GGUF)
```

## Tech stack

React Native 0.81 / Expo SDK 54 · TypeScript · llama.rn · @shopify/flash-list · Reanimated · expo-secure-store · Node.js + Express proxy · Vitest

## Getting started

**Requirements:** Node.js 20+, pnpm, an Android device or emulator, and an [NVIDIA API key](https://build.nvidia.com/) for cloud mode. The offline mode needs an Expo dev client build, because `llama.rn` is a native module and won't run in Expo Go.

```bash
git clone https://github.com/suhailult777/pocketclaw.git
cd pocketclaw
pnpm install
pnpm proxy:install
```

Create `server/.env`:

```env
NVIDIA_API_KEY=your-key
PORT=8787
DEFAULT_MODEL=nvidia/nemotron-3-nano-30b-a3b
MODEL_FALLBACK_QUALITY=nvidia/llama-3.3-nemotron-super-49b-v1.5
MODEL_FALLBACK_CAPACITY=nvidia/llama-3.1-nemotron-nano-8b-v1
JINA_API_KEY=            # optional, better web search
ZEROCLAW_GATEWAY_URL=    # optional, paired PC gateway
ZEROCLAW_WEBHOOK_SECRET= # optional
```

Run:

```bash
pnpm proxy:start         # proxy on http://127.0.0.1:8787
pnpm start               # Expo (cloud mode works in Expo Go)
pnpm start:dev           # dev client, needed for offline llama.rn mode
pnpm android:apk         # preview APK via EAS
```

In the app's **Settings**:
- Choose `nvidia-proxy` or `native`.
- On a physical phone, set the proxy host to your PC's LAN IP (not `localhost`).
- For native mode, pick a `.gguf` model file.

## Scripts

| Command | What it does |
|---|---|
| `pnpm typecheck` | TypeScript check |
| `pnpm test` | Typecheck + syntax check of the proxy |
| `pnpm test:unit` | Vitest unit tests |
| `pnpm test:integration` | Proxy integration test |

## Project structure

```
App.tsx
src/
  screens/   ChatScreen, SettingsScreen
  lib/       providerRouter, nvidiaProxyClient, nativeClient (llama.rn), tool executors
  context/   settings + persistence
  hooks/     streaming text, throttling
server/      Express proxy to NVIDIA NIM, tool calls, command policy
tests/       Vitest suites
plugins/     Expo config plugin for the native module
```

## Security

The command gateway executes on your PC. Keep it on a trusted local network, set `ZEROCLAW_WEBHOOK_SECRET`, and never expose the proxy or gateway ports to the internet.

## License

MIT
