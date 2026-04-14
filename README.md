# bun-webview

An agent skill for browser automation using **Bun.WebView** — Bun's built-in headless browser API introduced in [Bun v1.3.12](https://bun.sh/blog/bun-v1.3.12).

## What is Bun.WebView?

`Bun.WebView` is a native headless browser API built into Bun. It supports two backends:

| Backend | Description |
|---|---|
| `webkit` | macOS default. Uses system WKWebView. No existing sessions. |
| `chrome` | Connects to system Chrome via CDP. Inherits the default profile including cookies and existing logins. |

## Skill Contents

```
skills/
├── SKILL.md                  # Core reference — API, execution model, patterns
└── references/
    ├── patterns.md            # Code snippets for common tasks
    └── gotchas.md             # Known issues and workarounds (v1.3.12)
```

## Usage

This skill is intended to be loaded into an AI agent's skill system. Place the contents of `skills/` under `~/.claude/skills/bun-webview/` and your agent will use it when browser automation tasks arise.

### Quick example

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome', width: 1400, height: 900 });
  await v.navigate('https://example.com');
  for (let i = 0; i < 15; i++) { await new Promise(r => setTimeout(r, 1000)); if (!v.loading) break; }
  await new Promise(r => setTimeout(r, 2000));
  console.log(v.title, v.url);
"
```

### Connecting to an existing Chrome session

To reuse your existing Chrome cookies and logins, launch Chrome with the remote debugging port before running Bun.WebView:

```bash
open -a "Google Chrome" --args --remote-debugging-port=9222
```

Then use `backend: 'chrome'` — Bun will connect to the running instance automatically.

## Requirements

- [Bun](https://bun.sh) v1.3.12 or later
- macOS (for WebKit backend) or any platform with Google Chrome installed (for Chrome backend)

## Related

- [agent-browser](https://github.com/vercel-labs/agent-browser) — A Rust-based browser automation CLI for AI agents, also using Chrome via CDP
