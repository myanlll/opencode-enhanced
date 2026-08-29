# NOTICE, attribution and license compliance

This project is a derivative work. It combines three lineages:

## 1. opencode (upstream base)

- Repository: <https://github.com/anomalyco/opencode>
  (formerly `sst/opencode`; the repository was transferred, it is the same project)
- License: **MIT**
- Copyright (c) 2025 opencode
- Full license text preserved at: `LICENSES/opencode-MIT.txt`

Essentially all of the code in this repository is opencode's. This project is a
thin distribution on top of it, not a rewrite.

## 2. Kilo Code (browser automation approach)

- Repository: <https://github.com/Kilo-Org/kilocode>
- License: **MIT**
- Copyright (c) 2026 Kilo Code
- Copyright (c) 2025 opencode
- Full license text preserved at: `LICENSES/kilocode-MIT.txt`

Kilo Code is itself an opencode derivative (its own LICENSE preserves opencode's
copyright notice alongside its own).

The browser-automation feature in this repository was implemented after studying
Kilo Code's `BrowserAutomationService`
(`packages/kilo-vscode/src/services/browser-automation/browser-automation-service.ts`).

To be precise about what was and was not copied: Kilo Code's implementation is a
thin integration layer that registers Microsoft's official
[`@playwright/mcp`](https://github.com/microsoft/playwright-mcp) server with the
agent backend, passing `--headless` and `--browser chrome` flags based on user
settings. This repository implements the equivalent behaviour natively against
opencode's own MCP configuration layer. No Kilo Code source code was copied
verbatim; the design and the specific flag/defaults choices are theirs, and
attribution is given here accordingly.

Kilo Code's MIT license notice is preserved regardless, both to satisfy the MIT
notice condition and because attribution is simply the right thing to do.

## 3. Modifications original to this repository

- A tokens/second indicator in the TUI
- The `chat` agent mode preset
- The browser-automation integration described above
- Packaging, documentation, and build/distribution glue

## License of this combined work

The combined work is distributed under the **PolyForm Noncommercial License
1.0.0**, see `LICENSE`.

### Why this is permitted

The MIT license grants the right to "use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies", subject to one condition: the
copyright notice and permission notice must be preserved. It does not require
derivative works to be licensed under MIT. Relicensing a derivative work under
different (including more restrictive) terms is therefore permitted, provided
the original notices are preserved, which is what `LICENSES/` and this file do.

### What this does *not* do

It does not restrict upstream opencode or Kilo Code. Both remain MIT and can be
obtained from their own repositories and used commercially by anyone. The
noncommercial restriction applies only to this specific combined distribution
and the work original to it.

### Honest caveat

PolyForm Noncommercial 1.0.0 is a **source-available** license. It is *not*
OSI-approved and does not meet the Open Source Definition, because it restricts
the field of use (item 6 of the OSD, "No Discrimination Against Fields of
Endeavor"). Calling this project "open source" without qualification would be
inaccurate; "source available, free for noncommercial use" is the accurate
description.
