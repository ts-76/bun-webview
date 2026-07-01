---
name: bun-webview
description: Browser automation using Bun.WebView — Bun's built-in headless browser API. Use when the user needs to scrape web pages, automate authenticated sites, fill forms, extract DOM content, take screenshots, or run JavaScript in a browser context. Bun.WebView is a JavaScript API executed via `bun -e "..."` — not a CLI tool. Prefer this when Bun is available and the task involves authenticated browser sessions or DOM manipulation. With Chrome, assume an already-open Chrome CDP session and reuse it; never repeatedly launch fresh Chrome sessions.
---

# Bun.WebView

Bun v1.3.12+ ships native headless browser automation as a built-in API.

**Two backends, one API:**
- **WebKit** (macOS default) — zero dependencies, uses system WKWebView. No existing sessions.
- **Chrome** — connects to an already-running system Chrome via CDP. Inherits the open Chrome session, including cookies and existing logins.

## Chrome Session Reuse

**Default to reusing the already-open Chrome session. Do not repeatedly open Chrome.**

Before creating `new Bun.WebView({ backend: 'chrome' })`, verify that Chrome is already listening on the CDP port. If `http://127.0.0.1:9222/json/version` is unavailable, stop and tell the user Chrome needs to be opened once with `--remote-debugging-port=9222`; do not let Bun.WebView fall back to launching a fresh Chrome process.

```bash
bun -e "
  const cdp = await fetch('http://127.0.0.1:9222/json/version').catch(() => null);
  if (!cdp?.ok) {
    console.error('Chrome CDP is not available on 127.0.0.1:9222. Open Chrome once with --remote-debugging-port=9222, then rerun.');
    process.exit(1);
  }

  await using v = new Bun.WebView({ backend: 'chrome', width: 1400, height: 900 });
  await v.navigate('https://example.com');
  console.log(await v.evaluate('document.title'));
"
```

Use one `bun -e` process for a complete automation task whenever practical. Creating a separate `bun -e` process per step reconnects to Chrome and opens extra tabs; batch navigation, DOM inspection, form filling, clicking, and screenshots in the same script.

## Execution

Bun.WebView is a **JavaScript API**, not a CLI. Always run with `bun -e`:

```bash
bun -e "
  const cdp = await fetch('http://127.0.0.1:9222/json/version').catch(() => null);
  if (!cdp?.ok) process.exit(1);

  await using v = new Bun.WebView({ backend: 'chrome', width: 1400, height: 900 });
  await v.navigate('https://example.com');
  console.log(await v.evaluate('document.title'));
"
```

Use `bun -e` for all tasks — both quick inspections and multi-step automation. Use `await using` so the WebView/tab is disposed when done without closing the user's already-running Chrome session.

## Instance Reuse

**Create one instance per `bun -e` task and reuse it for all operations.**

`v.navigate()` reuses the same tab — there is no need to create a new `Bun.WebView` per URL or per step. Creating multiple instances within the same session means launching (or re-attaching to) Chrome multiple times, which adds startup overhead and makes the task slower.

```bash
# Correct — one instance, multiple navigations
bun -e "
  const cdp = await fetch('http://127.0.0.1:9222/json/version').catch(() => null);
  if (!cdp?.ok) process.exit(1);

  await using v = new Bun.WebView({ backend: 'chrome', width: 1400, height: 900 });

  for (const url of ['https://example.com', 'https://bun.sh']) {
    await v.navigate(url);
    for (let i = 0; i < 15; i++) { await new Promise(r => setTimeout(r, 1000)); if (!v.loading) break; }
    await new Promise(r => setTimeout(r, 2000));
    console.log(url, '->', v.title);
  }
"

# Wrong — new instance per URL (slow, unnecessary Chrome restarts)
# await using v1 = new Bun.WebView(...); await v1.navigate('https://example.com'); ...
# await using v2 = new Bun.WebView(...); await v2.navigate('https://bun.sh'); ...

# Wrong — separate bun -e invocations per step (opens extra tabs / reconnects repeatedly)
# bun -e "await using v = new Bun.WebView({ backend: 'chrome' }); await v.navigate(url1)"
# bun -e "await using v = new Bun.WebView({ backend: 'chrome' }); await v.navigate(url2)"
```

## Session / Authentication

```bash
bun -e "
  const cdp = await fetch('http://127.0.0.1:9222/json/version').catch(() => null);
  if (!cdp?.ok) process.exit(1);

  await using v = new Bun.WebView({ backend: 'chrome' });
  await v.navigate('https://app.example.com/dashboard');
  // Chrome profile cookies are inherited — no login needed
"
```

With `backend: 'chrome'`, behavior depends on whether Chrome is already running:

| Chrome state | Bun.WebView behavior |
|---|---|
| Running with `--remote-debugging-port=9222` | Connects to the **existing process**, opens a new tab. All logins and open tabs are shared. No cold start. |
| Running normally (no debug port) | May launch a **new Chrome process** using the default profile. Avoid this by running the CDP preflight before creating `Bun.WebView`. |

When connected to a running Chrome, you can see and interact with all currently open tabs via `v.cdp("Target.getTargets", {})`. This is the same as agent-browser's `--auto-connect`.

## Page Load

There is no `waitForLoad()`. Poll `v.loading`:

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome', width: 1400, height: 900 });
  await v.navigate('https://example.com');
  for (let i = 0; i < 15; i++) {
    await new Promise(r => setTimeout(r, 1000));
    if (!v.loading) break;
  }
  await new Promise(r => setTimeout(r, 2000)); // Wait for JS frameworks to settle
  console.log(v.title, v.url);
"
```

## DOM Inspection

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome' });
  await v.navigate('https://example.com');
  // ... wait ...

  // Always JSON.stringify for structured data — evaluate() always returns string
  const items = JSON.parse(await v.evaluate(\`
    JSON.stringify(
      [...document.querySelectorAll('a')].map(a => ({ text: a.textContent?.trim(), href: a.href }))
    )
  \`));
  console.log(items);
"
```

## Form Filling

For React/Vue-controlled inputs, use the JS property setter — `v.type()` is slow and often misses framework reactivity:

```bash
bun -e "
  // ... navigate, wait ...
  const result = await v.evaluate(\`
    (() => {
      const input = document.querySelector('input[name=\"email\"]');
      if (!input) return 'not_found';
      input.focus();
      const setter = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set;
      setter.call(input, 'user@example.com');
      input.dispatchEvent(new Event('input', { bubbles: true }));
      input.dispatchEvent(new Event('change', { bubbles: true }));
      return 'ok';
    })()
  \`);
"
```

## Clicking

`v.click(selector)` waits for the element to become actionable (visible, enabled, stable). If a button stays disabled after JS-injected input, click via `evaluate()`:

```bash
bun -e "
  // Native click — waits for actionability automatically
  await v.click('button[type=\"submit\"]');

  // Fallback — JS click when v.click() times out
  await v.evaluate(\`document.querySelector('[data-testid=\"submit\"]').click()\`);
"
```

## Screenshots

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome', width: 1400, height: 900 });
  await v.navigate('https://example.com');
  // ... wait ...
  const png = await v.screenshot({ format: 'png' });
  await Bun.write('page.png', png);
"
```

## API Reference

| Method / Property | Description |
|---|---|
| `v.url` | Current URL |
| `v.title` | Page title |
| `v.loading` | True while navigating |
| `v.navigate(url)` | Navigate to URL |
| `v.evaluate(expr)` | Execute JS — always returns `string` |
| `v.screenshot({format, quality})` | Capture PNG / JPEG / WebP |
| `v.click(x, y)` / `v.click(selector)` | OS-level click (`isTrusted: true`) |
| `v.type(text)` | Type into focused element (avoid for large content) |
| `v.press(key, {modifiers})` | Key press |
| `v.scroll(dx, dy)` / `v.scrollTo(selector)` | Scroll |
| `v.goBack()` / `v.goForward()` / `v.reload()` | Navigation |
| `v.resize(w, h)` | Resize viewport |
| `v.cdp(method, params)` | Raw CDP call |

For more patterns → `references/patterns.md`  
For known issues → `references/gotchas.md`
