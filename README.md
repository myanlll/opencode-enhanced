# opencode-enhanced

[![License: PolyForm Noncommercial](https://img.shields.io/badge/License-PolyForm%20Noncommercial-blue.svg)](LICENSE)
[![Built on opencode](https://img.shields.io/badge/built%20on-opencode-black)](https://github.com/anomalyco/opencode)
[![GitHub stars](https://img.shields.io/github/stars/myanlll/opencode-enhanced?style=social)](https://github.com/myanlll/opencode-enhanced)

|  |  |  |
|---|---|---|
| **Get started** | [Quick install](#install) | [BYOK setup](#bring-your-own-key-byok) |
| **The three additions** | [Tokens/second](#1-tokenssecond-indicator) | [`chat` mode](#2-chat-agent-mode) · [Browser automation](#3-browser-automation) |
| **Keeping it updated** | [Staying current with upstream](#staying-current-with-upstream) | [Troubleshooting](docs/opencode-enhanced.md) |
| **The fine print** | [License](#license) | [Attribution](NOTICE.md) |

Hello folks! This one started as a personal itch, not a grand plan. I run models
on my own hardware, a mix of local and remote self-hosted boxes, and
[opencode](https://github.com/anomalyco/opencode)
never told me how fast those models were actually running. "Feels slow today"
is not a metric, so I fixed that, then kept going and fixed two more things
while I was in there.

To be clear up front: this is a fork, not a rewrite. 99% of the code here is
opencode's, and all the credit for the hard part goes to them, I just bolted
three things onto it that I personally wanted:

| # | Addition | Why |
|---|---|---|
| 1 | **Tokens/second indicator** in the TUI | See real generation speed when running local models |
| 2 | **`chat` agent mode** | Use models that have no tool-calling support |
| 3 | **Browser automation** (Playwright + system Chrome) | Give the agent real browser control |

---

## 1. Tokens/second indicator

opencode shows you the response. It does not show you how *fast* it arrived.

That is fine when you are calling a hosted frontier model, where throughput is
roughly constant and not your problem. It is much less fine when you are running
your own inference, Ollama, vLLM, llama.cpp, LM Studio, a spare GPU box on the
network, because generation speed is the single number that tells you whether
your setup is healthy.

Quantisation level, context length, GPU/CPU split, whether the model actually
fit in VRAM or is quietly spilling into system RAM: all of these show up as
tokens/second, and almost nothing else surfaces them. A model that normally runs
at 27 tok/s dropping to 1.8 tok/s is not a subtle regression, but without a
readout you just experience it as "feels slow today".

This fork adds two things to the TUI:

- A **live** counter pinned above the input while the model is generating,
  showing token count, current tok/s, and elapsed time. It updates ~4×/second
  and is fixed in place, so it does not scroll away with the conversation.
- A **final** per-message figure appended to each completed assistant message's
  footer, so you can scroll back and compare.

The live figure estimates token count from streamed characters (÷4) until the
real usage numbers arrive, then switches to the exact value. So treat the
in-flight number as an indicator and the settled number as accurate.

No configuration. It is always on.

## 2. `chat` agent mode

Many capable local models are poor at, or entirely incapable of, tool calling.
Point opencode at one of those and you get a bad time: the agent tries to invoke
tools, the model emits malformed calls or hallucinates results, and the session
derails before you get an answer.

`chat` mode is a preset that removes the problem by removing the tools. Every
tool is disabled and every permission is set to `deny`, leaving a plain
conversational assistant with no file, shell, or network access.

Useful for:

- Models with no tool-calling support at all (Gemma 2, many finetunes)
- Models whose tool-calling is unreliable enough to be worse than nothing
- Just wanting to ask a question without an agent deciding to read your repo

Select it like any other agent mode. The definition lives in your
`opencode.json`, see [`docs/opencode-enhanced.md`](docs/opencode-enhanced.md) for the
exact block to copy.

## 3. Browser automation

An agent that can drive a real browser (navigate, click, fill forms, read
rendered pages) is the headline feature of tools like Google's Antigravity. You
do not need a closed-source IDE for it.

This fork can register Microsoft's official
[`@playwright/mcp`](https://github.com/microsoft/playwright-mcp) server
automatically, giving the agent browser tools with no manual MCP configuration.

Enable it with an environment variable:

```bash
OPENCODE_BROWSER_AUTOMATION=1 opencode
```

That is the whole setup. It uses your **system-installed Chrome** by default
rather than downloading a separate Chromium, and runs **headed** by default so
you can watch what the agent is doing.

| Variable | Default | Effect |
|---|---|---|
| `OPENCODE_BROWSER_AUTOMATION` | off | Master switch |
| `OPENCODE_BROWSER_HEADLESS` | `false` | Run with no visible window |
| `OPENCODE_BROWSER_SYSTEM_CHROME` | `true` | Use system Chrome instead of bundled Chromium |

Verify it is live:

```bash
$ OPENCODE_BROWSER_AUTOMATION=1 opencode mcp list
┌  MCP Servers
│
●  ✓ browser  connected
│      npx -y @playwright/mcp@latest --browser chrome
│
└  1 server(s)
```

If you already define a `browser` MCP server in your own config, yours wins and
this does nothing.

**Credit where it is due:** this approach is ported from
[Kilo Code](https://github.com/Kilo-Org/kilocode), which is itself an opencode
derivative and which implements its browser agent the same way. See
[`NOTICE.md`](NOTICE.md).

---

## Install

Requires [Bun](https://bun.sh) (the version in the root `package.json`
`packageManager` field) and a C toolchain for native modules.

```bash
git clone https://github.com/myanlll/opencode-enhanced.git
cd opencode-enhanced
bun install
bun run --cwd packages/opencode build --single
```

`--single` builds only for your current platform. The binary lands at:

```
packages/opencode/dist/opencode-<platform>-<arch>/bin/opencode
```

Put it somewhere on your `PATH`, or keep it out of the way and use a shell
wrapper, the wrapper approach is handy because it lets you pin environment
variables:

```bash
# ~/.zshrc
opencode() {
  OPENCODE_DISABLE_CHANNEL_DB=1 \
  OPENCODE_BROWSER_AUTOMATION=1 \
  "$HOME/.opencode-enhanced/opencode" "$@"
}
```

### macOS: re-sign after copying

If you copy the built binary to another location on macOS, **re-sign it**:

```bash
codesign --force --sign - /path/to/opencode
```

Without this the binary is killed on launch with **exit code 137** (SIGKILL) and
no error message. This affects Bun-compiled binaries generally, not just this
project. Full explanation in
[`docs/opencode-enhanced.md`](docs/opencode-enhanced.md#macos-exit-code-137).

## Bring your own key (BYOK)

opencode talks to any OpenAI-compatible endpoint via the
`@ai-sdk/openai-compatible` adapter, so local servers and hosted APIs are
configured identically. Example using a hosted OpenAI-compatible provider
alongside a local Ollama instance:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "hosted": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Hosted provider",
      "options": { "baseURL": "https://your-provider.example/v1" },
      "models": {
        "some-model-id": { "name": "Some Model" }
      }
    },
    "local": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Local Ollama",
      "options": { "baseURL": "http://127.0.0.1:11434/v1" },
      "models": {
        "qwen2.5-coder:32b": { "name": "Qwen2.5 Coder 32B" }
      }
    }
  }
}
```

Any OpenAI-compatible hosted provider works the same way, just swap the
`baseURL` and model id. This includes providers with a free tier, opencode's
own [provider docs](https://opencode.ai/docs/providers) list what's out there.

API keys go in opencode's credential store, **never in your config file or this
repo**:

```jsonc
// ~/.local/share/opencode/auth.json , git-ignored, chmod 600
{
  "hosted": { "type": "api", "key": "YOUR_KEY_HERE" }
}
```

More detail in [`docs/opencode-enhanced.md`](docs/opencode-enhanced.md).

## Staying current with upstream

Upstream opencode moves fast. Rebasing is cheap for the two actual code
changes: tokens/second touches one file, browser automation touches two, and
neither is a file upstream is likely to touch the same way.

```bash
git fetch origin
git rebase v<latest-tag>
bun install
bun run --cwd packages/opencode build --single
```

Two things in this fork are *not* cheap to rebase, be aware:

- **`README.md`** is fully rewritten here. Upstream also owns this file, so a
  future rebase will likely conflict on it, possibly more than once, since
  several commits in this fork touch it. Each time, resolve with:

  ```bash
  git checkout --theirs README.md
  git add README.md
  git rebase --continue
  ```

  **Important, and easy to get backwards:** during a rebase, `--ours` means
  the branch you are rebasing *onto* (upstream), and `--theirs` means the
  commit being replayed (this fork's own change). That is the opposite of
  what `--ours`/`--theirs` mean during a normal merge, and it is a common
  source of silently keeping the wrong version. This was verified by actually
  running the rebase against a simulated future upstream commit, not assumed.
- **The 21 translated `README.*.md` files were deleted** in this fork. If a
  future upstream release touches one of those, the rebase will pause on a
  modify/delete conflict. Resolution is `git rm <file>` to confirm the
  deletion and continue.

Neither is dangerous, both are `git status` telling you exactly where it
stopped, but don't expect the update to be a single command with zero
prompts.

## License

**PolyForm Noncommercial License 1.0.0**, free to use, modify, and share for
any noncommercial purpose. See [`LICENSE`](LICENSE).

To be accurate rather than flattering: PolyForm Noncommercial is
**source-available**, not OSI-approved open source, because it restricts field
of use. If you need a permissive, commercially-usable license, use
[upstream opencode](https://github.com/anomalyco/opencode) directly, it is MIT,
and this fork cannot and does not change that.

Both upstream projects are MIT and their notices are preserved in
[`LICENSES/`](LICENSES/). Attribution details are in [`NOTICE.md`](NOTICE.md).

## Credits

- [**opencode**](https://github.com/anomalyco/opencode), the actual coding
  agent this is built on. Nearly all credit belongs here.
- [**Kilo Code**](https://github.com/Kilo-Org/kilocode), the browser
  automation approach.
- [**Playwright MCP**](https://github.com/microsoft/playwright-mcp), Microsoft's
  MCP server doing the real browser work.

Upstream's original README is preserved as
[`README.opencode-upstream.md`](README.opencode-upstream.md).
