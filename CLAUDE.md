# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A LuCI (OpenWrt web UI) extension that adds "Subscribe URL" functionality to the existing **Podkop** proxy plugin. It fetches base64 subscription feeds, parses proxy links (vless/ss/trojan/hy2/socks), lists them in the Podkop config form, and applies a chosen config to Podkop or directly to Xray.

There is **no build system, no package manager, no test suite**. The repo is a set of source files plus an installer. UI strings and comments are mostly in Russian — keep that convention when editing user-facing text.

## Deployment / "running" it

`install.sh` (and `uninstall.sh`) are meant to run **on the OpenWrt router as root**, not locally. `install.sh` `wget`s each file from GitHub `main` into router paths, `chmod +x`es the CGI scripts, writes the ACL, and optionally installs `xray-core` + an `/etc/init.d/xray` init script. There is no way to fully exercise the code off-device; verification is manual in a router's LuCI UI under **Services → Podkop**.

Repo path → router path mapping (mirrors `files/`):
- `files/www/cgi-bin/*` → `/www/cgi-bin/*` (executable, POSIX `sh`)
- `files/www/luci-static/resources/view/podkop/*.js` → same under `/www/...` (mode 644)
- `files/usr/share/rpcd/acl.d/*.json` → same

Sanity-check shell before committing: `sh -n files/www/cgi-bin/<script>` (or `shellcheck` if available). The CGI scripts target BusyBox `sh` — no bashisms.

## Architecture

The plugin bolts onto Podkop's own `section.js` in two independent ways; both call into `subscribe.js`:

1. **Integrated (`section.js`)** — this repo ships a *modified copy* of Podkop's `section.js` that adds `"require view.podkop.subscribe as subscribeExt"` and builds the config form (`createSectionContent`). `install.sh` overwrites the router's `section.js` with this one (backing up the original to `section.js.backup`, using `contains_plugin_code` to avoid backing up an already-patched file).
2. **Non-invasive fallback (`subscribe-loader.js`)** — self-contained IIFE that waits for LuCI, monkey-patches `form.TypedSection.prototype.renderContents`, and also runs a `MutationObserver` to detect the Podkop form and inject Subscribe fields via raw DOM when the integrated path isn't present. Has its own duplicated fetch/render/style logic.

`subscribe.js` is the core module (~1800 lines). It `return baseclass.extend(EntryPoint)` and exports `enhanceSectionWithSubscribe(section)` + `injectSubscribeStyles()`. `enhanceSectionWithSubscribe` adds the `subscribe_url` field and per-mode "Получить" (Fetch) buttons to a Podkop section.

### The four proxy_config_type modes

`proxy_config_type` (a Podkop `ListValue`) drives everything via `o.depends(...)`. Each mode has its own fetch button, config-list DOM id, and click handler in `subscribe.js`:

| Mode | Applies to | Behavior |
|------|-----------|----------|
| `url` | Podkop `proxy_string` textarea | single select, one config |
| `outbound` | Xray via `podkop-xray-config` | single select, writes `/etc/xray/config.json`, restarts xray |
| `urltest` | Podkop `urltest_proxy_links` DynamicList | multi-select, auto-latency |
| `selector` | Podkop `selector_proxy_links` DynamicList | multi-select, manual |

XHTTP/REALITY configs are blocked in `url`/`urltest`/`selector` and only allowed in `outbound` (they need Xray). See `isXhttpConfig`.

### DOM targeting by section_id

LuCI renders form fields with ids like `cbid.podkop.<section_id>.<field>` (widget variant `widget.cbid.podkop...`). Because Podkop can have multiple sections, nearly every helper in `subscribe.js` is keyed by `section_id` extracted from the triggering element (`getSectionIdFromElement`, `findSubscribeInput`, config-list ids suffixed with `-<section_id>`). When adding UI, follow this per-section-id pattern or state leaks across sections.

### CGI backends (`files/www/cgi-bin/`)

All are POSIX `sh`, emit `Content-Type` + JSON, use `jshn.sh`/`jsonfilter`:

- **`podkop-subscribe`** — POST subscribe URL as body. `wget`s it with `User-Agent: v2rayNG` (required to get base64 links), base64-decodes, parses each line's protocol + `#title`, returns `{configs:[{url,title,protocol}]}`.
- **`podkop-xray-config`** — POST a `vless://` URL. Parses UUID/host/port/params by hand, generates a full Xray `config.json` (socks inbound on 10808 → vless outbound), writes `/etc/xray/config.json`, records the URL to `/tmp/podkop-xray-current-outbound`, restarts xray via `/etc/init.d/xray restart`. Only VLESS is generated here.
- **`podkop-configs-cache`** — GET/POST/DELETE per-section cache at `/tmp/podkop-subscribe-cache/<section_id>_<mode>.json`. Lets the UI restore fetched lists without re-fetching (`saveConfigsToCache`/`autoLoadCachedConfigs` in `subscribe.js`).
- **`podkop-current-outbound`** — GET, returns the contents of `/tmp/podkop-xray-current-outbound` (used to highlight the active outbound config).

The ACL (`files/usr/share/rpcd/acl.d/luci-app-podkop-subscribe.json`) grants read access to ubus `podkop.subscribe` `fetch`.

## Editing gotchas

- `section.js` here must stay a valid drop-in replacement for upstream Podkop's `section.js` — it's not purely ours. Changes to Podkop's own form fields belong upstream; keep additions confined to the subscribe integration.
- Two copies of the fetch/parse/render logic exist (`subscribe.js` and `subscribe-loader.js`). A behavior change often needs to be mirrored in both.
- Fetched config highlighting depends on the `/tmp` tracker/cache files, which are volatile (cleared on reboot) — don't assume they persist.
- When adding files, also add the corresponding `wget` + `chmod` block to `install.sh` and the matching removal in `uninstall.sh`; nothing is auto-discovered.
