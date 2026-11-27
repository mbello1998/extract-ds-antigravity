# URL acquisition

Read this only when the user gives a URL instead of a local folder. The goal is
a local `$SOURCE` folder that the normal workflow can copy. Stop as soon as one
route produces a page with the real visible content.

## Where `$SOURCE` goes

Create `$SOURCE` as `<slug>-source/` in the user's usual projects folder, or in
the current directory when there is none. `<slug>` is the URL's first host
label or last path segment, in kebab-case. `$OUTPUT` is then created beside it
as usual.

## Route 1: static fetch (try first)

Use `read_url_content` for a quick look, or save the raw HTML with
`run_command`. In PowerShell, type `curl.exe`, never `curl`:

```powershell
New-Item -ItemType Directory -Force "$SOURCE" | Out-Null
curl.exe -sL "$URL" -o "$SOURCE\index.html"
```

`read_url_content` may return the page as text or Markdown. Judge the content
with it, but save the file with `curl.exe`.

Treat the page as **static** when the HTML body contains the visible headings
and copy of the page. Then fetch its requisites once and continue on the fast
path:

```powershell
wget.exe -q -E -k -p -nH -e robots=off -P "$SOURCE" "$URL"
```

Windows usually has no `wget.exe` (and `wget` in PowerShell is an alias for
`Invoke-WebRequest`). If `wget.exe` is missing, keep the `curl.exe` HTML and
leave its asset URLs absolute.

## Detect an app shell

Treat the page as an **app shell** when most of these hold:

- the HTML is small (a few KB) and the body is an empty root such as
  `<div id="root"></div>` or `<div id="app"></div>`;
- the visible headings and copy are absent from the HTML;
- it loads one large bundle (hundreds of KB or more) that contains the whole
  hosting platform (editor, dashboard, auth, analytics), not the page;
- the bundle reads page content at runtime from a backend (Supabase, Firebase,
  a CMS API).

Detect the shell by these symptoms, not by a list of hostnames. Builders such
as aura.build, Lovable, Bolt, and similar hosted editors usually work this way.

Do not download the platform bundle as `$SOURCE`. Do not read its backend
tables or RPCs. Use Route 2.

## Route 2: rendered capture in the browser

Call `browser_subagent` with one precise task: open the URL, wait until the
visible headline exists in the DOM, run the given JavaScript, and report its
return value exactly. Paste the script into the task text. Ask for a short
return value (counts, lengths, `ok`), never the page source: long results get
cut. Then try the captures in this order.

### 2a. `iframe[srcdoc]` (best result)

Many builders sandbox the published page in an iframe whose `srcdoc` holds the
original page source before any script ran. Check:

```js
JSON.stringify([...document.querySelectorAll('iframe')].map(f => ({
  srcdoc: (f.getAttribute('srcdoc') || '').length,
  src: f.getAttribute('src'),
  w: f.offsetWidth, h: f.offsetHeight
})));
```

Pick the largest visible iframe with a long `srcdoc` and save it as a file:

```js
const f = [...document.querySelectorAll('iframe')]
  .sort((a, b) => (b.getAttribute('srcdoc') || '').length - (a.getAttribute('srcdoc') || '').length)[0];
const blob = new Blob([f.getAttribute('srcdoc')], { type: 'text/html' });
const a = Object.assign(document.createElement('a'),
  { href: URL.createObjectURL(blob), download: 'extract-ds-source.html' });
document.body.appendChild(a); a.click(); a.remove();
'ok ' + f.getAttribute('srcdoc').length;
```

The file normally lands in the user's Downloads folder. A repeat download may
be named `extract-ds-source (1).html`, so move the newest match:

```powershell
$f = Get-ChildItem "$env:USERPROFILE\Downloads\extract-ds-source*.html" |
  Sort-Object LastWriteTime -Descending | Select-Object -First 1
Move-Item -LiteralPath $f.FullName "$SOURCE\index.html"
```

If no file appears there, ask `browser_subagent` where the browser saved the
download, then move it from that folder. Antigravity's browser may use its own
profile with a different download folder.

Inspection scripts in the browser must return counts or hostnames, not full
URLs with query strings. Such results may hold cookie or query data, and the
browser tool may block or truncate them.

Both aura.build variants observed so far (plain HTML projects and React
projects) expose a full `srcdoc`, so 2a is the normal route there. This capture keeps the original scripts and animations,
so continue with the normal runtime preflight.

#### Clean the `srcdoc` capture

Do this once on the copy in `$OUTPUT`, never on `$SOURCE`:

1. **Strip the platform runtime.** Builders inject their own scripts at the
   top of `<head>`, before the page's first `<meta charset>`: session data,
   `parent.postMessage` bridges, storage shims, token firewalls, CMS loaders
   (on aura.build: `script[data-session]`, `#aura-supabase-token-firewall`).
   They only work inside the platform. Keep everything from the first
   `<meta charset>` onward. Keep platform helpers placed later that are plain
   page behavior, such as a video autoplay controller.
2. **Add `<!doctype html>`.** An iframe `srcdoc` always renders in standards
   mode, even without a doctype. A saved file without one opens in quirks mode
   and the layout shifts.
3. **Keep the `<html>` attributes** (`lang`, `class="scroll-smooth"` and
   similar) when you rewrite the head.
4. **Clear baked runtime state.** The platform may serialize the `srcdoc` after
   scripts ran. Look for flags that block re-initialization, such as
   `data-initialized="true"` or text a script already wrote (a typewriter
   showing one letter). Remove the flag so the script starts again.
5. **Neutralize server routes.** Root-relative navigation links (`/work`,
   `/about`) break under `file://`. Point them to `#`.

### 2b. Same-origin or blob iframe

When the iframe has `src` instead of `srcdoc`, open that `src` directly if it is
a normal URL and restart from Route 1. When it is same-origin or `blob:`, read
`f.contentDocument` and save it as in 2c.

### 2c. Post-render DOM (fallback)

When there is no usable iframe, save the rendered document. Collect readable
stylesheet rules so the styles survive, and remove the platform bundle so the
copy does not try to re-render from the backend:

```js
const doc = document.documentElement.cloneNode(true);
doc.querySelectorAll('script[type="module"], script[src*="/assets/"], link[rel="modulepreload"]')
  .forEach(n => n.remove());
let css = '';
for (const s of document.styleSheets) {
  try { css += [...s.cssRules].map(r => r.cssText).join('\n') + '\n'; } catch (e) {}
}
const style = document.createElement('style');
style.textContent = css;
doc.querySelector('head').appendChild(style);
const html = '<!doctype html>\n' + doc.outerHTML;
const blob = new Blob([html], { type: 'text/html' });
const a = Object.assign(document.createElement('a'),
  { href: URL.createObjectURL(blob), download: 'extract-ds-source.html' });
document.body.appendChild(a); a.click(); a.remove();
'ok ' + html.length;
```

Keep cross-origin stylesheet `<link>` tags (Google Fonts, CDNs) as they are.
Move the file to `$SOURCE/index.html`.

Tell the user, in Portuguese, before Phase 1:

> Este site é um app que monta a página em tempo de execução, sem um HTML
> original disponível. Capturei a página já renderizada. O visual fica fiel, mas
> animações de entrada e de scroll podem não funcionar, porque o runtime do app
> não vem junto. Quer seguir assim?

Continue only after the user agrees. Do not rebuild the missing animations
unless the user asks.

## Assets

- Keep public storage URLs (for example
  `https://<project>.supabase.co/storage/v1/object/public/...`) and CDN URLs
  absolute. They load from `file://`.
- Download an asset only when it fails to load from `file://` and the user asked
  for an offline package.
- Fix root-relative asset paths (`/assets/x.png`) by prefixing the site origin.
  Leave root-relative navigation links (`/about`, `/work`) alone; they are
  secondary routes, not assets.

## Hard limits

- Never call the platform's backend directly: no Supabase `rest/v1` or
  `rpc/...` requests, no Firebase or GraphQL queries, no private JSON endpoints,
  even with a public anon key found in a bundle. Capture only what the page
  itself renders for a normal visitor.
- Do not log in, bypass a paywall, or capture pages that require an account.
- When a site blocks every route, report which route failed and why, and ask
  the user for the local project files or an exported HTML.
