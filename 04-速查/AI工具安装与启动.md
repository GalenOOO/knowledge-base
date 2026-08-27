# AI工具安装与启动

> 本文档目的：回头查常用 AI 编程工具的安装与启动命令。信息截至 2026-08。

多数 npm 包需要 Node 22+。

## DeepSeek Harness

```
npx @deepseek-ai/dsh web
```

默认 Web UI：`http://127.0.0.1:3080`。需要 Node `^22.19.0` 或 `>=24`。

全局安装后：`dsh web`。

```
npm i -g @deepseek-ai/dsh
```

## Claude Code

macOS / Linux / WSL：

```
curl -fsSL https://claude.ai/install.sh | bash
```

Windows PowerShell：

```
irm https://claude.ai/install.ps1 | iex
```

启动：`claude`。

## Codex CLI

macOS / Linux：

```
curl -fsSL https://chatgpt.com/codex/install.sh | sh
```

Windows PowerShell：

```
powershell -ExecutionPolicy ByPass -c "irm https://chatgpt.com/codex/install.ps1 | iex"
```

或：`npm i -g @openai/codex`。启动：`codex`。

## Cursor Agent CLI

macOS / Linux / WSL：

```
curl https://cursor.com/install -fsS | bash
```

Windows PowerShell：

```
irm 'https://cursor.com/install?win32=true' | iex
```

启动：`agent`。

## Gemini CLI

```
npx @google/gemini-cli
```

全局安装后：`gemini`。

```
npm i -g @google/gemini-cli
```

## OpenCode

```
curl -fsSL https://opencode.ai/install | bash
```

或：`npm i -g opencode-ai@latest`。启动：`opencode`。

## GitHub Copilot CLI

```
npm i -g @github/copilot
```

启动：`copilot`。

## Qwen Code

```
npm i -g @qwen-code/qwen-code@latest
```

启动：`qwen`。需要 Node 22+。

## Kimi Code CLI

macOS / Linux：

```
curl -LsSf https://code.kimi.com/install.sh | bash
```

Windows PowerShell：

```
irm https://code.kimi.com/install.ps1 | iex
```

或已有 uv：`uv tool install --python 3.13 kimi-cli`。启动：`kimi`。

## Pi Agent

```
curl -fsSL https://pi.dev/install.sh | sh
```

或：`npm i -g --ignore-scripts @earendil-works/pi-coding-agent`。启动：`pi`。

## MiniMax CLI

```
npm i -g mmx-cli
```

登录：`mmx auth login --api-key sk-xxxxx`。需要 Node 18+。

## Hermes Agent

macOS / Linux / WSL2：

```
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

Windows PowerShell：

```
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
```

启动：`hermes`。选模型：`hermes model`。

## OpenClaw

macOS / Linux / WSL2：

```
curl -fsSL https://openclaw.ai/install.sh | bash
```

Windows PowerShell：

```
iwr -useb https://openclaw.ai/install.ps1 | iex
```

或已有 Node（`22.22.3+` / `24.15+` / `25.9+`）：

```
npm i -g openclaw@latest --allow-scripts=openclaw
openclaw onboard --install-daemon
```

启动：`openclaw`。安装脚本会直接进入 onboarding。
