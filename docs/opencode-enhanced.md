# opencode-enhanced, configuration, build notes, and troubleshooting

Companion document to the [README](../README.md).

---

## Configuration

### `chat` agent mode

Add to `~/.config/opencode/opencode.json`. All tools disabled, all permissions
denied, a plain conversational assistant for models that cannot tool-call.

```jsonc
{
  "agent": {
    "chat": {
      "mode": "primary",
      "description": "Plain chat mode, no tool calling required",
      "prompt": "You are a helpful, direct conversational assistant. Respond naturally and concisely to the user's message. You have no tools and no code/file access, this is a plain chat conversation.",
      "tools": {
        "bash": false, "read": false, "edit": false, "glob": false,
        "grep": false, "webfetch": false, "task": false, "todowrite": false,
        "websearch": false, "lsp": false, "skill": false
      },
      "permission": {
        "read": "deny", "edit": "deny", "glob": "deny", "grep": "deny",
        "list": "deny", "bash": "deny", "task": "deny",
        "external_directory": "deny", "todowrite": "deny", "webfetch": "deny",
        "websearch": "deny", "question": "deny", "doom_loop": "deny"
      }
    }
  }
}
```

Verify it registered:

```bash
$ opencode agent list
build (primary)
plan (primary)
chat (primary)     # <-- here
...
```

### Browser automation

| Variable | Default | Effect |
|---|---|---|
| `OPENCODE_BROWSER_AUTOMATION` | off | Registers the `browser` MCP server |
| `OPENCODE_BROWSER_HEADLESS` | `false` | Adds `--headless` |
| `OPENCODE_BROWSER_SYSTEM_CHROME` | `true` | Adds `--browser chrome` |

Implementation: `packages/opencode/src/config/config.ts` injects a `browser`
entry into the resolved MCP config; flags are declared in
`packages/core/src/flag/flag.ts`. An explicit `browser` server in user config
takes precedence.

Requires `npx` on `PATH`. The first run downloads `@playwright/mcp`, so allow
extra time, the configured startup timeout is 60s.

### Environment variables inherited from upstream

`OPENCODE_DISABLE_CHANNEL_DB=1` forces the shared main database
(`opencode.db`) instead of a per-channel database. This is **stock upstream
behaviour**, implemented in `packages/core/src/database/database.ts`, not
something this fork adds. Useful if you build your own binary and do not want
sessions split across a separate channel-specific database file.

`OPENCODE_DISABLE_AUTOUPDATE=1` (or `"autoupdate": false` in config) suppresses
the update check. See [Update prompts](#update-prompts-on-a-custom-build) below.

---

## Build notes

```bash
bun install
bun run --cwd packages/opencode build --single
```

Useful flags for `build`:

| Flag | Effect |
|---|---|
| `--single` | Build only for the current platform (otherwise: all targets) |
| `--skip-install` | Skip re-installing cross-platform native deps, much faster on rebuilds |
| `--sourcemaps` | Emit linked sourcemaps |

Publishing is gated behind the `OPENCODE_RELEASE` environment variable
(`packages/script/src/index.ts`). It is unset by default, so a normal local
build will never attempt to upload a GitHub release.

### Version strings

The build derives its version from the current git branch when no explicit
version is set. On a non-`latest` channel this produces:

```
0.0.0-<branch>-<UTC timestamp>
```

e.g. `0.0.0-opencode-enhanced-202608291941`. This is expected for fork builds.

---

## Troubleshooting

### macOS: exit code 137

**Symptom.** The binary builds fine and runs from its build directory, but after
you copy it elsewhere it dies instantly and silently:

```
$ ~/.opencode-enhanced/opencode --version
$ echo $?
137
```

Exit 137 is `128 + 9`, SIGKILL. There is no error message anywhere.

**Diagnosis.** The copy is byte-identical to the original:

```
$ shasum -a 256 dist/opencode-darwin-x64/bin/opencode ~/.opencode-enhanced/opencode
3f96d1425b464a05e2d54aa5fa3cacb01ee65a59cd60e33ca0f44eaee323e5a8  dist/.../opencode
3f96d1425b464a05e2d54aa5fa3cacb01ee65a59cd60e33ca0f44eaee323e5a8  ~/.opencode-enhanced/opencode
```

and yet:

```
$ codesign -v ~/.opencode-enhanced/opencode
/Users/you/.opencode-enhanced/opencode: invalid signature (code or signature have been modified)
In architecture: x86_64
```

Note that the *original* reports the same "invalid signature", Bun-compiled
binaries carry an ad-hoc signature that does not survive relocation. The
original keeps working because macOS cached a successful validation for it at
its original path when the build's smoke test executed it. The copy has no such
cache entry, fails validation, and is killed by the kernel.

**Fix.** Re-sign ad-hoc after copying:

```bash
codesign --force --sign - /path/to/opencode
```

```
/path/to/opencode: replacing existing signature
$ /path/to/opencode --version
0.0.0-opencode-enhanced-202608291941
```

Make this part of your install step; it is required on every copy.

### Update prompts on a custom build

A locally built binary reports a version like `0.0.0-<branch>-<timestamp>`.
Because `0.0.0` sorts below every published release, the updater will *always*
consider your build out of date and keep offering an update.

Two things worth knowing:

1. **It will not overwrite your build.** `Installation.method()`
   (`packages/opencode/src/installation/index.ts`) detects the install method by
   probing package managers, e.g. running `brew list --formula opencode`,
   not by looking at the path of the running binary. If you also have a
   packaged opencode installed, the updater targets *that* copy, not your custom
   build. Additionally, automatic upgrades only run when the version delta is a
   `patch`; a `0.0.0` → `1.x.y` delta is a major bump, so the code path only ever
   emits a notification and returns.

2. **You can silence it.** Either set `OPENCODE_DISABLE_AUTOUPDATE=1`, or add
   `"autoupdate": false` to your config.

### Rebasing onto a new upstream release

The fork's diff is intentionally tiny, so this is usually uneventful:

```bash
git fetch origin --tags
git rebase v<latest-tag>
bun install          # dependencies do drift between releases
bun run --cwd packages/opencode typecheck
bun run --cwd packages/opencode build --single
```

Files this fork touches, and thus the only plausible conflict sites:

| File | Change |
|---|---|
| `packages/tui/src/routes/session/index.tsx` | tokens/second indicator (~88 lines) |
| `packages/core/src/flag/flag.ts` | three browser automation flags |
| `packages/opencode/src/config/config.ts` | browser MCP server injection |

`session/index.tsx` is the one that sees real upstream churn. If the patch stops
applying cleanly, the intent is small enough to reapply by hand: a
`LiveTokenStatus` component rendered just above the prompt input, plus a
`tokensPerSecond` memo appended to the assistant message footer.

Always run `typecheck` before `build`, it fails much faster and catches
renamed upstream APIs, which is the realistic failure mode after a large jump.
