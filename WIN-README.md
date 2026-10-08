# ClaudePace for Windows: plan

Status: **Planned, not implemented.** This document is self-contained. The only external reference is the macOS [README.md](README.md), which describes the feature set and budget math to match.

## Goal

A Windows 10/11 version of ClaudePace with full parity with the macOS app described in [README.md](README.md), plus a CLI. It reads Claude Code's session logs, prices them, and tracks spend against a monthly budget from a tray icon.

## Decisions so far

| Topic | Decision |
|---|---|
| Core language | Rust |
| UI | Tauri 2 (WebView2) with an HTML/CSS/SVG popup |
| Layout | One Cargo workspace, no sidecar processes |
| CLI | Second binary in the same workspace, sharing the core crate |
| Installer | MSI (WiX, built by Tauri) |
| Signing | Self-signed certificate (see [Signing](#signing-self-signed)) |
| macOS app | Not part of this work |

Why Rust over Go: Tauri is Rust, so a Go core would need a sidecar `.exe` and an IPC layer, two toolchains, and bitmap/label data passed back across the process boundary. A Rust core is called directly from Tauri commands.

## Workspace layout

```
claudepace-win/
  Cargo.toml                 # workspace
  crates/
    core/                    # claudepace-core: no UI, no OS-specific code
      src/budget.rs          # budget and pace math
      src/pricing.rs         # built-in price table + LiteLLM merge
      src/scanner.rs         # log scanning, dedupe, caching
      src/export.rs          # json, csv by day/model/project/session
    cli/                     # claudepace (scan, export, check)
  app/                       # Tauri app
    src-tauri/               # tray, window, commands, settings, tray-icon renderer
    ui/                      # popup frontend
```

Crates likely needed: `serde`, `serde_json`, `chrono` (local time zone), `walkdir`, `reqwest` (price fetch), `dirs`, `clap` (CLI), `tiny-skia` or `image` with a font crate (tray label), `tauri`, `tauri-plugin-clipboard-manager`.

## Core

Behavior to preserve exactly, as described in [README.md](README.md) and restated here:

- **Log roots:** `$CLAUDE_CONFIG_DIR` (comma-separated) if set, otherwise `~/.config/claude` and `~/.claude`, each with `/projects/**/*.jsonl`. On Windows `~` is `%USERPROFILE%`.
- **Parsing:** only lines containing `"usage"` are decoded. Skip `<synthetic>` models. Timestamps are ISO8601 with or without fractional seconds. Lines may end in `\r`.
- **Dedupe:** Claude Code logs a response several times while streaming, so keep the costliest copy per `message.id` + `requestId`. Entries missing either id are always counted.
- **Cache pricing:** 5-minute and 1-hour cache writes are priced separately (`ephemeral_1h_input_tokens`).
- **Project name:** last component of `cwd`. Split on both `\` and `/`, since Windows logs contain backslashes. Fall back to the project directory name.
- **Titles:** `custom-title` beats `ai-title` for session names.
- **File cache:** reuse parsed entries while a file's size and mtime are unchanged; skip files not modified since the window start.
- **Prices:** built-in fallback table, merged with LiteLLM's public price list, refreshed daily, cached in `%LOCALAPPDATA%\dev.meain.claudepace\litellm-prices.json`.
- **Budget math:** usable = budget x (1 - reserve%); daily allowance = usable / days in month; expected spend = allowance x completed days; days ahead = (expected - spent) / allowance, truncated toward zero for the label; today's target = (usable - spent before today) / days left including today; "left today" = target - spent today. "Today" is derived from per-day totals so it rolls over at midnight.
- **Months** follow the local time zone.

Testing: write Rust unit tests for the budget math (month progress, pace, today's target, rollover at midnight, reserve handling), and add scanner tests with fixture `.jsonl` files (CRLF lines, duplicate streaming copies, both cache TTLs, missing ids, Windows-style `cwd`).

## CLI

- `claudepace scan` prints this month's spend by model.
- `claudepace export --format json|csv [--by day|model|project|session]`.
- `claudepace check` runs the budget math checks.

## Tauri app

- **Tray icon:** Windows tray icons cannot show text, so the label (`+2d`, `-1d`, `42%`, `$62`) is rendered into a 32x32 icon bitmap by the Rust side and applied with the tray's `set_icon` after every refresh. Tooltip carries the longer text. Needs a legibility check at 100%, 150% and 200% display scaling, and in light and dark taskbars.
- **Popup:** a borderless, always-on-top window that opens next to the tray on click and closes when it loses focus. Same layout as described in [README.md](README.md):
  - pace-aware "left today"
  - month progress bar with reserve and where spend should be by today
  - pace, spent this month, today and remaining
  - daily-spend sparkline against the daily allowance, last month's daily spend as a grey line, a % change vs the same point last month, and hover tooltips
  - breakdown by model, project or session with a monthly/daily toggle and expandable rows
  - toolbar: settings, refresh (Ctrl+R), export, quit (Ctrl+Q)
- **Sparkline:** hand-written SVG, no chart library.
- **Export:** share button copies JSON or CSV to the clipboard.
- **Settings:** budget (default 2000), reserve % (default 10), menu bar mode (days / % used / $ left), stored as JSON in `%APPDATA%\dev.meain.claudepace\settings.json`.
- **Refresh:** every 5 minutes, scan on a background thread.
- **Single instance:** a second launch focuses the existing one (`tauri-plugin-single-instance`).
- **Launch at login:** optional, via `tauri-plugin-autostart`. Not part of parity, so it's an optional extra.

## Signing (self-signed)

A self-signed certificate gives a valid signature but no trust: Windows shows "Unknown publisher" and SmartScreen warns until the certificate is trusted on that machine. This is fine for personal or team use and not for public distribution.

One-time certificate creation (PowerShell):

```powershell
$cert = New-SelfSignedCertificate -Type CodeSigningCert `
  -Subject "CN=ClaudePace (self-signed)" `
  -KeyAlgorithm RSA -KeyLength 3072 -HashAlgorithm SHA256 `
  -CertStoreLocation Cert:\CurrentUser\My -NotAfter (Get-Date).AddYears(3)
Export-Certificate -Cert $cert -FilePath claudepace-signing.cer          # public part, share this
Export-PfxCertificate -Cert $cert -FilePath claudepace-signing.pfx `
  -Password (Read-Host -AsSecureString "pfx password")                   # private, never commit
```

Signing: `signtool sign /fd sha256 /tr http://timestamp.digicert.com /td sha256 /f claudepace-signing.pfx /p <password> <file>`. Sign the app `.exe`, the CLI `.exe`, then the `.msi`. Tauri can sign the binaries during bundling through `bundle.windows.signCommand`. Always timestamp, so signatures stay valid after the certificate expires.

Making the signature trusted on a machine: import `claudepace-signing.cer` into **Trusted Root Certification Authorities** and **Trusted Publishers** (admin, `certutil -addstore`). Only do this on machines you control.

CI: keep the `.pfx` (base64) and its password as GitHub secrets. The signing step runs only when the secrets exist, so forks and local builds still produce an unsigned MSI. Do not commit the `.pfx`; add `*.pfx` to `.gitignore`.

Upgrade path: if the app is later shared publicly, swap the certificate for Azure Trusted Signing or an OV/EV certificate with no change to the build, only the `signtool` arguments.

## Packaging and release

- `tauri build --bundles msi` on `windows-latest` (WiX is fetched by Tauri). Prerequisites on the build machine: Rust stable (MSVC), Node (if the UI uses a bundler), Visual Studio Build Tools.
- New GitHub Actions job `release-windows` attaching `ClaudePace-<version>-x64.msi`.
- Version number starts at 0.2.0 to match the macOS app.

## Milestones

1. **Core:** Rust workspace; port budget, pricing, scanner and export with tests passing against fixtures.
2. **CLI:** `scan`, `export`, `check`.
3. **Tray:** tray icon with rendered label, settings file, refresh loop, three label modes.
4. **Popup:** full layout, sparkline with last-month overlay and hover, breakdown tabs with daily/monthly toggle, export to clipboard, shortcuts.
5. **Installer and signing:** MSI build, self-signed signing step, CI job, install/uninstall test on a clean Windows 11 VM.
6. **Docs:** a Windows section in the main README, including how to trust the self-signed certificate.

## Risks and open questions

- **Tray label legibility:** a 16-32 px bitmap holds about 3-4 characters, so `$62` fits but `-$124` may not. The design needs a shrink or abbreviation rule.
- **Popup positioning:** the tray's screen position isn't always available, so placement near the taskbar needs care across taskbar positions and multiple monitors.
- **WebView2:** preinstalled on Windows 11 and most up-to-date Windows 10, but the installer should bootstrap it if missing.
- **Windows log differences:** Claude Code on Windows may lay out or name project directories differently. Confirm with real logs on a Windows machine before finalizing the scanner tests.
- **Price source:** the LiteLLM URL is unauthenticated GitHub raw content, and corporate proxies may block it. The built-in fallback table must stay current, and proxy settings may need to be honored.
- **Self-signed trust:** every installing machine needs the certificate imported first, or users see an "Unknown publisher" warning.
- **Feature drift:** the macOS app may gain features, so keep a short checklist mapping each feature in [README.md](README.md) to its Windows counterpart.
