# AGENTS.md

Personal portfolio / CV site (French) for Simon Foussat-Ferré. Static, no build system, no dependencies, no tests, no CI.

## What this repo actually is

- Tracked files: `README.md` and `index.html` only. `main` is the only branch.
- Deployed as a GitHub **user** Pages site: repo is `komafeu007.github.io`, served at `https://komafeu007.github.io/` from the root of `main`. Pushing to `main` publishes the site.
- There is **no source project in this repo** and no bundler/CLI. `index.html` is the generated artifact.

## Critical: `index.html` is a generated bundle, not source

Do not hand-edit the visible markup. The `<body>` only contains "Unpacking…" loading placeholders (lines ~21–23). Everything real lives in three `<script>` blocks that a bootstrap script (lines ~25–373) decodes at runtime:

- `<script type="__bundler/template">` (line 388) — a JSON **string** containing the actual page HTML: all text, inline CSS, the `<x-dc>` design-component markup, and an inline `<script type="text/x-dc">` component-logic class.
- `<script type="__bundler/manifest">` (line 376) — JSON map of uuid → asset. Holds the gzip+base64 `dc-runtime` engine, React 18 + ReactDOM 18 (inlined), and two woff2 fonts (`compressed` flag marks gzip; fonts are raw base64).
- `<script type="__bundler/ext_resources">` and `__bundler/page_order` — external-resource markers and nested-page list (currently empty).

The runtime inlines/decompresses via `DecompressionStream('gzip')`. Editing anything outside the template/manifest strings has no effect on the rendered page.

## Editing content

To change what the page shows, edit the HTML inside the **template** script, then re-serialize it as a JSON string.

Gotcha: the decoded template contains a literal `</script>` (from the inner `text/x-dc` tag). The stored field escapes `/` as `\u002F` (e.g. `<\/title>`) so it does not prematurely close the outer `<script>`. A naive `json.dumps` emits a literal `</script>` and corrupts the page. Keep the `/` (and `<`) of `</...>` escapes when re-serializing, e.g. replace `</` with `<\/` in the dumped string.

Decode helper:

```bash
python3 - <<'PY'
import re, json, gzip, base64
html = open('index.html').read()
tpl = json.loads(re.search(r'<script type="__bundler/template">\s*(.*?)\s*</script>', html, re.S).group(1))
open('/tmp/opencode/template.html', 'w').write(tpl)          # editable page HTML
man = json.loads(re.search(r'<script type="__bundler/manifest">\s*(.*?)\s*</script>', html, re.S).group(1))
for k, v in man.items():
    data = gzip.decompress(base64.b64decode(v['data'])) if v.get('compressed') else base64.b64decode(v['data'])
    open(f'/tmp/opencode/{k}', 'wb').write(data)              # decoded assets
PY
```

Preferred path if available: regenerate the bundle with the original design/AI tool that produced this format (the dc-runtime source says "GENERATED … Rebuild with `bun build.ts` (see the package README)", but that package is not in this repo). Prefer regeneration over hand-editing the escaped JSON.

## Verifying changes

No test/lint/typecheck exists. Always open `index.html` in a browser and confirm the content renders (a corrupted template/manifest shows an error banner or a stuck "Unpacking…" screen). A quick sanity check that JSON still parses:

```bash
python3 -c "import re,json; h=open('index.html').read(); [json.loads(re.search(r'<script type=\"__bundler/%s\">\s*(.*?)\s*</script>'%t,h,re.S).group(1)) for t in ('manifest','template')]; print('ok')"
```

## Conventions / notes

- Site language is French; keep content French unless asked otherwise.
- Fonts: IBM Plex Mono and a second woff2, loaded via `fonts.gstatic.com` `@font-face` in the template; avoid adding new external hosts.
- Contact link in template: `mailto:simonferre33@gmail.com`; also links to github.com and linkedin.com.
- Design-canvas sizing is set by `<meta name="hz:canvas-*">` (1360×2750) and `hz:slide-selector`.
