# Gotchas & Known Issues

## `backend: "chrome"` vs WebKit

- **WebKit** (macOS default): launches a fresh browser with no existing sessions
- **Chrome**: uses the system Chrome profile → cookies and logins are inherited
- Use `backend: "chrome"` whenever authentication is needed

## No `headless: false`

As of Bun v1.3.12, only headless mode is supported. Do not pass `headless: false` — it throws an error.

## No `waitForLoad()` or `waitForNetworkIdle()`

Always use the polling pattern:

```bash
for (let i = 0; i < 15; i++) {
  await new Promise(r => setTimeout(r, 1000));
  if (!v.loading) break;
}
await new Promise(r => setTimeout(r, 2000)); // extra wait for SPA hydration
```

## `evaluate()` Always Returns `string`

Even when the JS expression produces a number, boolean, or object, `evaluate()` returns a string. Use `JSON.stringify` / `JSON.parse`:

```bash
# Wrong
const count = await v.evaluate("document.querySelectorAll('li').length");
// → "5" (string)

# Correct
const count = parseInt(await v.evaluate("document.querySelectorAll('li').length"), 10);
const data  = JSON.parse(await v.evaluate("JSON.stringify({ ... })"));
```

## `v.click(selector)` Actionability Timeout

`v.click()` waits for the element to be visible, enabled, and stable. Buttons that are only enabled after framework-reactive input may never satisfy this condition despite being present in the DOM.

**Workaround**: click via `evaluate()` instead.

## `v.type()` Misses Framework Reactivity

`v.type()` simulates keystrokes one character at a time. React/Vue event handlers often don't fire correctly with synthetic keystroke events, and it's very slow for long text.

**Use the JS property setter** with `dispatchEvent('input')` and `dispatchEvent('change')` instead (see SKILL.md → Form Filling).

## Escaping in `bun -e` Strings

Inside a `bun -e "..."` shell string, backtick-quoted template literals are safe. Use `\`` for the JS template literal and escape inner double-quotes as `\"`:

```bash
bun -e "
  const x = await v.evaluate(\`document.querySelector('h1')?.textContent\`);
"
```

## `v.scrollTo(selector)` Requires a Visible Element

`v.scrollTo()` times out if the selector doesn't match a currently visible element. Verify selectors first with `v.evaluate()`.

## One Chrome Instance Per Bun Process

Multiple `new Bun.WebView()` calls within one `bun -e` share a single Chrome subprocess; each creates a new tab. Navigation calls in sequence all operate in that shared context.

## CDP Events Are Not Delivered (v1.3.12)

The blog states CDP events should arrive as `MessageEvent` via `addEventListener('message')`, but in practice 0 events are received. This blocks:
- Video recording via `Page.startScreencast`
- Network request interception (`Network.requestWillBeSent`)
- Real-time DOM mutation observation

**Workaround for recording**: poll `v.screenshot()` in a loop (practical limit ~2fps).

## `screenshot()` Returns a `Blob`, Not `Uint8Array`

`v.screenshot()` returns a `Blob`. Use `Bun.write(path, blob)` directly — do not try to access `.byteLength` or iterate it.

## File Upload: Use CDP, Not `v.click()`

There is no `v.uploadFile()` method. Use `DOM.setFileInputFiles` via `v.cdp()`:

```ts
const doc = await v.cdp("DOM.getDocument", {});
const node = await v.cdp("DOM.querySelector", { nodeId: doc.root.nodeId, selector: "input[type=file]" });
await v.cdp("DOM.setFileInputFiles", { nodeId: node.nodeId, files: ["/path/to/file.png"] });
```

## No Electron / Native App Support

`backend: { type: "chrome", path, argv }` can point to a custom Chromium-based browser, but there is no way to connect to an already-running Electron app's CDP port. For Electron automation, use agent-browser instead.

## No Accessibility Tree Snapshot

agent-browser's `snapshot` command provides an accessibility tree with element refs (`@e1`, `@e2`…). Bun.WebView has no equivalent — use `v.evaluate()` with `document.querySelectorAll()` to locate elements instead.
