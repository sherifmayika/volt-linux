# Volt for Linux

A lightweight, drop-in Electron alternative. Run your existing JavaScript or TypeScript Electron app on roughly a fifth of the RAM, in a tenth of the installer size, with an `npm install` and one alias.

> **Status:** Working Linux port of [nehemiyawicks/volt](https://github.com/nehemiyawicks/volt). Tested on Arch-based distros (instantOS). See [`GOALS.md`](GOALS.md) for upstream milestones, [`PLAN.md`](PLAN.md) for the strategy, and [`CHANGELOG-LINUX.md`](CHANGELOG-LINUX.md) for every change versus upstream.

This fork makes Volt build and run natively on Linux using **WebKitGTK** through `wry` and `tao`, ahead of the upstream v0.2 milestone. Both backends (webkit + chromium) remain on the roadmap; this fork ships webkit.

---

## Why

Electron ships a full Chromium and Node.js runtime with every app. 150 to 350 MB of RAM per app just to sit idle. Tauri, Wails, and Electrobun solve the RAM problem but require rewriting your backend in Rust, Go, or a modified Bun entrypoint. Electron still wins because the switching cost keeps it entrenched.

Volt's bet: **the compat layer is the killer feature.** You keep your `main.ts`, your IPC handlers, your `BrowserWindow` code. You lose 150 MB. The target is any Electron app, not a specific one; see [`docs/hardest-apps.md`](docs/hardest-apps.md) for the ladder of apps that get tested on every release.

Volt ships two backends, picked per-app via `manifest.engine`:

- **`engine: "webkit"` (default)** — wry + WebKit. ~40 MB RAM per app. Best for most Electron apps. **This is what the Linux port currently ships.**
- **`engine: "chromium"` (v0.9)** — Chromium via CDP. Match Electron on RAM per single app. Required for VS Code and apps that use `<webview>`, V8 snapshots, or CDP. Not yet available on Linux.

| | Electron | Tauri | Electrobun | Volt (webkit) | Volt (chromium) |
|---|---|---|---|---|---|
| Idle RAM (single window) | 150-350 MB | 30-50 MB | ~35 MB | **~40 MB** | ~290 MB |
| Installer size per app | 100+ MB | <10 MB | ~12 MB | **<15 MB** | <15 MB + system Chromium |
| Backend language | JS / TS | Rust | Bun (partial rewrite) | **JS / TS** | JS / TS |
| Runs an unmodified Electron `main.ts` | yes (native) | no | no | **yes** | **yes** |
| Renders VS Code correctly | yes | no | no | no | **yes** |
| Uses `import { app, BrowserWindow } from 'electron'` unchanged | yes | no | no | **yes** | **yes** |

---

## Linux status

What works today on Linux, verified on instantOS with WebKitGTK 2.48:

| Feature | Status |
|---|---|
| Window creation, sizing, positioning, focus | ✅ |
| WebView rendering (WebKitGTK, embedded as native GTK widget) | ✅ |
| IPC — `ipcRenderer.invoke` / `send` / `on`, `ipcMain.handle` | ✅ |
| `shell.openExternal` (via `xdg-open`) | ✅ |
| Notifications (`notify-rust` over D-Bus) | ✅ |
| Clipboard — read, write, paste outside the app | ✅ |
| Open / save file dialogs (`zenity`) | ✅ |
| Global shortcuts | ✅ |
| DevTools (`webContents.openDevTools()`) | ✅ |
| Multi-window | ✅ |
| Application menu bar (GTK) | ⚠️ depends on desktop — see Known issues |
| Tray icons (`tray-icon` + Ayatana Indicator) | ⚠️ requires extension on GNOME |

Not on Linux yet: `engine: "chromium"` backend, V8 snapshots, `<webview>` tag.

---

## Install system dependencies

Volt links against GTK 3, WebKitGTK 4.1, and libsoup 3. It needs Rust 1.75+, Node 20+, and a handful of runtime helpers. **Distro `rustc` and `nodejs` are typically too old** — use `rustup` and `nvm` to get current versions.

### Arch Linux / Manjaro / EndeavourOS / CachyOS / instantOS

```bash
# Toolchains (rustup + nvm recommended over distro packages)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source "$HOME/.nvm/nvm.sh" && nvm install --lts

# System libraries
sudo pacman -S --needed \
    base-devel webkit2gtk-4.1 gtk3 libsoup3 \
    zenity xdg-utils

# Tray icon support (optional, AUR)
yay -S --needed libayatana-indicator
```

### Debian / Ubuntu / Mint / Pop!_OS / Fedora / openSUSE / Void / Gentoo / NixOS

**→ See [`SETUP.md`](./SETUP.md)** for per-distro commands, version caveats, and troubleshooting. It covers 8 distro families and the WebKitGTK 4.0-vs-4.1 trap on older Ubuntu releases.

---

## What ships today

`v0.1` on Linux contains:

- A monorepo scaffold (Rust host + npm workspace)
- Strategy docs: [`PLAN.md`](PLAN.md), [`GOALS.md`](GOALS.md)
- `@volt/electron-compat` with typed `app`, `BrowserWindow`, `ipcMain`, `ipcRenderer`, `dialog`
- A Rust host binary (`volt-core`) that opens a native GTK window via [`wry`](https://github.com/tauri-apps/wry) and [`tao`](https://github.com/tauri-apps/tao)
- A `volt` CLI (`init | dev | build`) with JS and TS templates
- A quick-start example, working end-to-end on Linux

---

## Migration in three steps

Both JavaScript and TypeScript projects follow the same shape.

**1. Install**

```bash
npm install @volt/electron-compat @volt/cli
```

**2. Add a `tsconfig.json` path (TS only, for type resolution)**

```json
{
  "compilerOptions": {
    "paths": {
      "electron": ["node_modules/@volt/electron-compat"]
    }
  }
}
```

JavaScript projects don't need this step. `volt dev` symlinks `node_modules/electron` to `@volt/electron-compat` before spawning your app, so bare `import ... from 'electron'` resolves at runtime in both languages.

**3. Run**

```bash
npx volt dev
```

Your existing `main.js` or `main.ts` runs unchanged: `app.whenReady()`, `new BrowserWindow(...)`, `ipcMain.handle(...)`.

---

## Building from source

```bash
git clone https://github.com/sherifmayika/volt-linux.git
cd volt-linux

# 1. Rust host — first build is 60-90 min, later builds are seconds
cargo build --release -p volt-core

# 2. JavaScript workspace
npm install
npm run build

# 3. Quick-start demo
npm run dev:example
```

Releases: pushing a tag `vX.Y.Z` triggers a GitHub Actions workflow that builds `volt-core` for Linux x86_64, macOS (arm64 + x86_64), and Windows x86_64, and attaches binaries to a Release.

---

## Linux-specific changes vs upstream

This fork diverges from `nehemiyawicks/volt` in four places. Full patch in [`CHANGELOG-LINUX.md`](./CHANGELOG-LINUX.md).

1. **Native WebView embedding** — `WebViewBuilder::new(&window).build()` → `WebViewBuilder::new_gtk(vbox)`. Fixes the blank-window bug on Wayland, XWayland, and some GPU/driver combos. `wry`'s default Linux path creates a nested X11 child surface that doesn't composite under a wayland session.
2. **File dialogs** — `rfd::FileDialog` opens a nested GTK main loop that crashes the app inside tao's event-loop callback. On Linux we call `zenity` in a subprocess instead. Requires `zenity` installed.
3. **Clipboard ownership** — `arboard::Clipboard` must outlive the write on X11/Wayland, or the clipboard manager's ownership is released and external pastes fail. The handle now lives in `AppState`.
4. **Menu bar** — `Menu.setApplicationMenu(...)` now attaches to existing GTK windows via `muda::Menu::init_for_gtk_window`, and to new windows at creation time. Menu bar visibility depends on the desktop environment (see below).

---

## Known issues on Linux

### Menu bar is invisible on GNOME

GNOME 4x hides per-window GTK menu bars by default. KDE, XFCE, Cinnamon, and MATE display them normally. On GNOME, use the keyboard accelerator (`Ctrl+J` in the quick-start demo) instead, or enable "Menu Bar" in `gnome-tweaks` under Top Bar.

### Tray icon doesn't appear on GNOME

Vanilla GNOME has no tray. Install the [AppIndicator and KStatusNotifierItem Support](https://extensions.gnome.org/extension/615/appindicator-support/) extension, and make sure `libayatana-indicator` is installed (AUR on Arch).

### Blank window with NVIDIA

Disable the DMABUF renderer:

```bash
WEBKIT_DISABLE_DMABUF_RENDERER=1 npm run dev:example
```

If that doesn't help, also disable accelerated compositing and enable explicit-sync workaround:

```bash
__NV_DISABLE_EXPLICIT_SYNC=1 \
WEBKIT_DISABLE_DMABUF_RENDERER=1 \
WEBKIT_DISABLE_COMPOSITING_MODE=1 \
npm run dev:example
```

If a combination works, export it permanently in your shell profile.

### `Gdk-CRITICAL ... gdk_window_thaw_toplevel_updates`

Harmless GTK teardown warning. Appears on successful runs. Ignore.

### `Error: write EPIPE` on shutdown

The JS child tries to write to a pipe whose Rust endpoint has closed. Noise, not a bug. Close the app with the window's X button (not `Ctrl+C`) to avoid it.

### `Failed to call ... org.freedesktop.portal.Secret` / `puffin.cc` errors

Chromium internals inside WebKitGTK looking for a Flatpak portal. Only matters if you're running under Flatpak. To suppress:

```bash
mkdir -p ~/.cache/volt
export XDG_CACHE_HOME=$HOME/.cache/volt
```

---

## Repository layout

```
crates/volt-core/          Rust host binary (wry + tao)
packages/electron-compat/  Drop-in replacement for require('electron')
packages/volt-cli/         volt init | dev | build
examples/quick-start/      Minimal working app
docs/                      Migration guide, API compat matrix
PLAN.md                    Strategy and architecture
GOALS.md                   Milestones with done-when criteria
SETUP.md                   Per-distro install commands (8 families)
CHANGELOG-LINUX.md         Every change vs nehemiyawicks/volt
```

---

## Contributing

Volt is early. The most valuable contributions right now:

- Migrating a real Electron app on Linux and filing an issue for every API that's missing or broken
- Implementing missing APIs against the [Electron docs](https://www.electronjs.org/docs/latest); copy the signature verbatim
- Extending the compat matrix in [`docs/compat.md`](docs/compat.md)
- Testing the install path on a distro other than Arch — add your distro to the "Tested" column in [`SETUP.md`](./SETUP.md)
- Fixing the GNOME menu-bar and tray gaps (upstream `muda`/`tray-icon` limitations, but workarounds are welcome)

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for ground rules.

---

## License

[Apache-2.0](LICENSE). See [`NOTICE`](NOTICE) for third-party attributions.
