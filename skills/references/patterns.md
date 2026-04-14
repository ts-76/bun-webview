# Patterns

> Each snippet below shows a standalone example with its own `await using v = new Bun.WebView(...)`.
> When combining multiple patterns in one `bun -e`, declare the instance **once at the top** and reuse `v` throughout — do not create a new instance per step.

## Check Auth / Detect Redirect

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome', width: 1400, height: 900 });
  await v.navigate('https://app.example.com/protected');
  for (let i = 0; i < 15; i++) {
    await new Promise(r => setTimeout(r, 1000));
    if (!v.loading) break;
  }
  await new Promise(r => setTimeout(r, 2000));

  if (v.url.includes('login') || v.url.includes('sign_in')) {
    console.error('Not authenticated. Log in to Chrome first.');
    process.exit(1);
  }
  console.log('Authenticated. Current page:', v.title);
"
```

## Extract a List of Items

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome' });
  await v.navigate('https://example.com/list');
  // ... wait for load ...

  const items = JSON.parse(await v.evaluate(\`
    JSON.stringify(
      [...document.querySelectorAll('.item')]
        .map(el => ({ name: el.querySelector('.name')?.textContent?.trim(), href: el.querySelector('a')?.href }))
        .filter(i => i.name)
    )
  \`));
  console.log(JSON.stringify(items, null, 2));
"
```

## Scrape Multiple Pages

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome', width: 1400, height: 900 });
  await v.navigate('https://example.com/index');
  // ... wait ...

  const links = JSON.parse(await v.evaluate(\`
    JSON.stringify([...document.querySelectorAll('a.item')].map(a => ({ name: a.textContent?.trim(), href: a.href })))
  \`));

  const results = {};
  for (const { name, href } of links) {
    await v.navigate(href);
    for (let i = 0; i < 10; i++) {
      await new Promise(r => setTimeout(r, 1000));
      if (!v.loading) break;
    }
    await new Promise(r => setTimeout(r, 2000));

    results[name] = await v.evaluate(\`document.querySelector('main')?.innerText?.trim() ?? ''\`);
  }

  console.log(JSON.stringify(results, null, 2));
"
```

## Fill and Submit a Form

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome', width: 1400, height: 900 });
  await v.navigate('https://example.com/form');
  // ... wait ...

  // Fill textarea (framework-safe)
  const filled = await v.evaluate(\`
    (() => {
      const ta = document.querySelector('textarea[name=\"body\"]');
      if (!ta) return 'not_found';
      ta.focus();
      const setter = Object.getOwnPropertyDescriptor(HTMLTextAreaElement.prototype, 'value').set;
      setter.call(ta, 'My comment text here');
      ta.dispatchEvent(new Event('input', { bubbles: true }));
      ta.dispatchEvent(new Event('change', { bubbles: true }));
      return 'ok:' + ta.value.length;
    })()
  \`);
  console.log('Filled:', filled);
  await new Promise(r => setTimeout(r, 500));

  // Submit — try JS click if v.click() times out
  await v.evaluate(\`document.querySelector('[type=\"submit\"]').click()\`);
  await new Promise(r => setTimeout(r, 3000));
  console.log('Submitted. URL:', v.url);
"
```

## Inspect DOM to Find Selectors

Use this to discover the right selectors before writing automation:

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome' });
  await v.navigate('https://example.com/page');
  // ... wait ...

  // Find all buttons with their text and data attributes
  console.log(await v.evaluate(\`
    JSON.stringify(
      [...document.querySelectorAll('button, [role=\"button\"]')]
        .map(b => ({ text: b.innerText?.trim(), testid: b.dataset.testid, type: b.type }))
        .filter(b => b.text || b.testid)
    )
  \`));

  // Find all inputs
  console.log(await v.evaluate(\`
    JSON.stringify(
      [...document.querySelectorAll('input, textarea, select')]
        .map(el => ({ tag: el.tagName, name: el.name, id: el.id, testid: el.dataset.testid, placeholder: el.placeholder }))
    )
  \`));
"
```

## Take Screenshot

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome', width: 1400, height: 900 });
  await v.navigate('https://example.com');
  for (let i = 0; i < 15; i++) { await new Promise(r => setTimeout(r, 1000)); if (!v.loading) break; }
  await new Promise(r => setTimeout(r, 2000));

  const png = await v.screenshot({ format: 'png' });
  await Bun.write('screenshot.png', png);
  console.log('Saved screenshot.png');
"
```

## Read from a CodeMirror Editor

```bash
bun -e "
  await using v = new Bun.WebView({ backend: 'chrome' });
  await v.navigate('https://example.com/editor');
  // ... wait ...

  const content = await v.evaluate(\`
    (() => {
      const cm = document.querySelector('.CodeMirror');
      if (cm?.CodeMirror) return cm.CodeMirror.getValue();
      const ta = document.querySelector('textarea.editor, textarea[name*=\"content\"]');
      return ta?.value ?? '';
    })()
  \`);
  console.log(content);
"
```
