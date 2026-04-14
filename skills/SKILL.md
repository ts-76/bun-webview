---
name: bun-webview
description: Browser automation using Bun.WebView — Bun's built-in headless browser API. Use when the user needs to scrape web pages, automate authenticated sites, fill forms, extract DOM content, take screenshots, or run JavaScript in a browser context. Bun.WebView is a JavaScript API executed via `bun -e "..."` — not a CLI tool. Prefer this when Bun is available and the task involves authenticated browser sessions or DOM manipulation.
---

# Bun.WebView

Bun v1.3.12+ ships native headless browser automation as a built-in API.

**Two backends, one API:**
- **WebKit** (macOS default) — zero dependencies, uses system WKWebView. No existing sessions.
- **Chrome** — connects to system Chrome via CDP. Inherits the default profile including cookies and existing logins.

## Execution

Bun.WebView is a **JavaScript API**, not a CLI. Always run with `bun -e`:

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome', width: 1400, height: 900 });
  await v.navigate('https://example.com');
  console.log(await v.evaluate('document.title'));
"
```

Use `bun -e` for all tasks — both quick inspections and multi-step automation. Use `await using` so the browser closes automatically when done.

## Instance Reuse

**Create one instance per `bun -e` session and reuse it for all operations.**

`v.navigate()` reuses the same tab — there is no need to create a new `Bun.WebView` per URL or per step. Creating multiple instances within the same session means launching (or re-attaching to) Chrome multiple times, which adds startup overhead and makes the task slower.

```bash
# Correct — one instance, multiple navigations
bun -e "
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
```

## Session / Authentication

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome' });
  await v.navigate('https://app.example.com/dashboard');
  // Chrome profile cookies are inherited — no login needed
"
```

With `backend: 'chrome'`, behavior depends on whether Chrome is already running:

| Chrome state | Bun.WebView behavior |
|---|---|
| Running with `--remote-debugging-port=9222` | Connects to the **existing process**, opens a new tab. All logins and open tabs are shared. No cold start. |
| Running normally (no debug port) | Launches a **new Chrome process** using the default profile. Cookies are inherited but open tabs are not shared. Cold start applies. |

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
