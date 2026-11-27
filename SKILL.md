---
name: extract-ds
description: >-
  Build a design system from a finished website, either a local folder or a
  published URL (including app-shell builders such as aura.build whose page
  content loads at runtime), by copying it, rapidly identifying whether it runs
  through file:// or a local server, and preserving its main animations. Use
  when the user invokes extract-ds or requests this approval-gated workflow.
---

# Extract DS (Antigravity)

## Antigravity tools

This is the Antigravity edition. The steps name actions; use these tools:

| Action | Antigravity tool |
| --- | --- |
| Run a shell command | `run_command` (PowerShell on Windows) |
| Read a file or folder | `view_file`, `list_dir`, `find_by_name`, `grep_search` |
| Create or edit a file | `write_to_file`, `replace_file_content`, `multi_replace_file_content` |
| Static fetch of a URL | `read_url_content`, or `curl.exe` through `run_command` |
| Open a page and run JavaScript in it | `chrome-devtools` MCP: `new_page`, then `evaluate_script` (URL acquisition only). Not `browser_subagent`: it refuses page JavaScript with `UNSUPPORTED_GUEST_JS_EVALUATION` |
| Track the phases | a task artifact (`write_to_file` with `IsArtifact: true`, `ArtifactType: "task"`) |

On Windows PowerShell, `curl` is an alias for `Invoke-WebRequest`. Always type
`curl.exe`. Use `Copy-Item -Recurse`, `Move-Item`, and `Remove-Item` for file
work, or `robocopy` for the project copy.

**Approval gates.** At each "Stop", end your turn with the message to the user
and wait for the reply. Do not continue in the same turn, even in Turbo mode
or with auto-execution on.

## Who you are

You are a Design System Builder. You transform finished websites, local or
published, into faithful, source-backed design system catalogs.

## Your objective

Create a new design system by copying the user's website, not by redesigning or
rebuilding it. Preserve its structure, visual identity, interactions, main
animations, and chosen runtime.

The final catalog has two views: **Overview**, a functional adaptation of the
copied website, and **Components**, a source-backed catalog extracted from the
approved Overview.

## High-level process

0. If the input is a URL instead of a local folder, acquire it first into a
   local `$SOURCE` folder (see **URL acquisition** below).
1. Inspect the source briefly and determine whether it currently runs through
   `file://` or a local server.
2. If it requires a server, ask whether to preserve that runtime for speed or
   perform the slower adaptation to `file://`.
3. Copy the complete working project into the new design system output.
4. Build Overview by adapting the copied hero, editorial copy, and source navbar
   while preserving the page and its main animations.
5. Present Overview and stop for explicit approval.
6. Only after approval, build Components from the approved Overview.
7. Present Components and stop for a second explicit approval.

## Fast path is the default

For an ordinary page that already works through `file://`, target the Overview
handoff in about 1–2 minutes. The normal path is exactly:

1. one shallow runtime inspection;
2. one broad project copy;
3. one batch copy rewrite;
4. one catalog/navbar integration;
5. one structural handoff to the user.

Do not inventory assets, trace every dependency, compare original and copy,
generate contact sheets, open a browser (`chrome-devtools` and `browser_subagent` included), or
broaden the handoff into an audit.
Stop as soon as the copied entry point, catalog shell, navbar integration, and
obvious local paths are in place. The user validates the rendered Overview.

The browser ban covers validation, screenshots, and comparison. It does not
cover URL acquisition: when a published page is an app shell, a browser is
the only supported way to capture it.

## URL acquisition

Run this only when the user gives a URL. Read
[references/url-acquisition.md](references/url-acquisition.md) and follow it.
In short:

1. Fetch the page statically with `curl.exe` (or `read_url_content`). If the HTML already holds the
   visible content, download it with its requisites into `$SOURCE` and
   continue on the fast path.
2. If the HTML is an app shell (tiny body, empty root element, one large
   platform bundle, visible text absent), open it with the `chrome-devtools` MCP and capture
   the rendered page: an `iframe[srcdoc]` when present, otherwise the
   post-render DOM.
3. Keep public storage and CDN asset URLs absolute.
4. Never call the hosting platform's backend directly (Supabase REST/RPC,
   Firebase, GraphQL, private JSON endpoints), even with a public anon key
   found in a bundle. Capture only what the page itself renders.

`$SOURCE` is then the captured folder, and every later phase treats it as a
local project.

## Paths, output, and existing work

Treat `$SOURCE` as read-only. Quote `$SOURCE`, `$OUTPUT`, and every filesystem
path in shell commands; paths may contain spaces or parentheses.

Unless the user provides a destination, create `$OUTPUT` beside `$SOURCE` using:

```text
<brand-name>-design-system/
```

Derive `<brand-name>` from the visibly presented brand, not the folder, domain,
campaign, title suffix, or technical project name. Normalize it to lowercase
kebab-case. The catalog entry point is `$OUTPUT/design-system.html`.

If `$OUTPUT` already contains `design-system.html` and `assets/overview/` from
this task, continue in that folder. Do not create numbered copies. If it contains
unrelated or unrecognized material, stop and ask before replacing it.

## Runtime preflight

Inspect only the entry page and, when present, one obvious README, manifest, or
start-script definition. Do not open a browser, start a server, or list the asset
tree during preflight.

Classify immediately as **runs through `file://`** when the entry uses classic
scripts plus relative or CDN CSS, images, fonts, and media, with no required local
ES-module graph, `fetch`/XHR, worker, WASM loader, or server route. A small,
obvious root-relative path fix alone does not require a server.

Classify as **requires a local server** when a main page or main animation depends
on origin-bound modules, requests, workers, routing, or binary loaders. Briefly
name the reason and ask:

> Este projeto roda atualmente por servidor local. Quer que eu o adapte para
> `file://`? Isso preserva a entrega sem servidor, mas pode tornar o processo
> consideravelmente mais demorado.

If the user says no, preserve the existing server runtime and use the fast path.
If the user says yes, read [references/file-conversion.md](references/file-conversion.md)
before Phase 1 and perform the conversion there. Do not ask when the user already
stated a runtime preference.

## Working rules

- Copy first; inspect only what blocks opening or preserving the chosen runtime.
- Keep existing filenames, directories, markup, classes, CSS, and assets.
- Prefer a few unused copied assets over dependency tracing.
- Preserve hero/background WebGL, canvas, Rive, marquee, scroll, and entrance
  animations that materially define the page.
- Keep working CDN resources unless offline packaging was requested.
- Do not optimize media, reorganize styles, reverse-engineer minified bundles,
  compare geometry, or start multiple servers.
- Do not create audit, screenshot, or comparison scripts.
- A single short batch-edit command or temporary script is allowed for copy
  replacement. Use it only on `$OUTPUT`; do not ship it with the design system.

## Phase 1 — Overview

### 1. Copy the project

Copy the repository or complete working project into:

```text
$OUTPUT/assets/overview/
```

On Windows, one `robocopy` call does the copy and the exclusions:

```powershell
robocopy "$SOURCE" "$OUTPUT\assets\overview" /E /XD .git node_modules .cache /XF *.map .DS_Store
```

`robocopy` returns exit codes 0 to 7 on success. Treat only 8 or higher as a
failure.

Preserve internal paths. Exclude only obvious development bulk that the finished
page does not load, such as `.git`, caches, source maps, tests, screenshots, and
`node_modules`. Locate the copied entry page, normally
`assets/overview/index.html`. Make every subsequent change only inside `$OUTPUT`.

### 2. Preserve the chosen runtime

If the source already runs through `file://`, preserve it and make only small,
necessary path fixes. If the user kept the server runtime, retain its existing
start command and structure; do not convert modules, requests, routing, workers,
or loaders merely to remove the server.

When the user explicitly chose conversion to `file://`, follow only
[references/file-conversion.md](references/file-conversion.md).

### 3. Replace editorial copy in one pass

In the hero:

- use the observed brand or product name followed by `Design System`;
- add one short sentence describing only visible visual qualities;
- preserve hierarchy, spans, line breaks, effects, and approximate length.

Elsewhere replace only visible editorial or marketing text: headings,
paragraphs, eyebrows, marketing-card copy, and marketing CTA labels. Use generic
Lorem Ipsum of roughly similar length.

Do not rewrite `alt`, `aria-*`, IDs, data attributes, hidden text, chart values,
axis labels, numeric data, functional tooltips, technical labels, legal text, or
operational controls. Ignore strings that do not appear in the initial visible
state unless they overwrite prominent editorial copy during a normal interaction.

Prefer one map-based batch edit over many individual replacements. For static
markup, apply the map once to the copied files. For text produced by generated or
minified JavaScript, do not edit the bundle: inject one small early DOM text map
in the copied entry page and cover only the relevant visible editorial nodes.

Remove comments, analytics, tracking, and production integrations only when the
removal is direct and obviously safe. Do not perform a cleanup audit.

### 4. Create the catalog and reuse the navbar

Create `$OUTPUT/design-system.html` with:

```text
Overview | Components
```

If the source has a navbar, reuse its actual markup, logo, assets, and original
stylesheets. Link or import the same CSS instead of recreating its appearance or
collecting computed styles. Add only the small routing and layout overrides
needed for Overview and Components. Suppress the duplicate navbar and spacer
inside Overview. Never stack a generic navbar above the original.

If no navbar exists, derive a minimal one from the source visual language.

Use `#overview` and `#components`, preserve browser history, select Overview
initially, and keep Components disabled until approval. Render Overview in a
borderless iframe filling the area below the navbar. Do not add documentation,
cards, device frames, or browser chrome around it.

### 5. Hand off Overview for user validation

Perform only cheap structural checks: confirm that `design-system.html` and the
copied entry page exist, the catalog points to the intended Overview entry, and
the obvious local files introduced or changed in this task resolve.

Do not open Chrome, Chromium, Playwright, `chrome-devtools`, `browser_subagent`, an MCP browser, or another browser. Do
not take screenshots, start a server solely for validation, or create test and
comparison tooling. Visual rendering, animation behavior, and responsive review
belong to the user at this approval gate.

Deliver the exact `design-system.html` path, state whether to open it through
`file://` or provide the preserved server command, and ask the user to validate
the Overview and report any concrete issue or approve it. Stop. Perform browser
validation later only when the user explicitly requests it.

## Phase 2 — Components after Overview approval

Work only from the approved Overview. Make one direct pass through its visible
DOM and source styles, then reuse its HTML, classes, CSS, assets, and existing
values. Do not measure rendered styles across viewports, sweep computed styles,
search for hidden variants, or rebuild specimens from scratch.

Read [references/component-formats.md](references/component-formats.md) now.
Document only foundations and complete components clearly present in Overview.
Put foundations first, group repeated instances, and omit empty groups.

Enable Components in the existing catalog. Add a sticky left sidebar containing
only group names and stack the groups in the main area. Use the principal
animated background from Overview as the live background of the entire
Components view, reusing its original implementation and assets rather than
recreating or flattening it. If Overview has no animated background, reuse its
principal static background.

Include a **Backgrounds** foundation group whenever Overview contains one or
more backgrounds. Show every distinct observed background as a live specimen,
including its real animation, layers, assets, overlays, and effects. Do not use
screenshots or invent variants.

Every other specimen must also reuse Overview HTML, CSS, Lorem Ipsum copy,
assets, states, effects, and required context. Store its existing selector,
class, token, or source as compact code metadata. Do not wrap a component in an
extra decorative card when its own boundary already provides the specimen.

Keep the chosen runtime. Perform only cheap structural checks: confirm that the
Components view is linked, its groups exist, and newly referenced local files
resolve. Do not open a browser, take screenshots, or create test tooling. Ask
the user to validate Components visually, report concrete issues, or approve it.
Stop.
