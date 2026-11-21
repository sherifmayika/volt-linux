# Volt — Linux setup guide

Step-by-step install instructions for every major Linux distribution family.
The end goal is the same on all distros: a working `cargo build --release -p volt-core`
plus a system `zenity` binary.

If your distro isn't listed, use the **Generic / from-source** section at the bottom.

---

## 0. Prerequisites at a glance

Volt needs **three** groups of packages:

| Group | What | Why |
|---|---|---|
| Language toolchains | Rust 1.75+, Node.js 20+ | compile the Rust host and JS workspace |
| System libraries | GTK 3, WebKitGTK 4.1, libsoup 3, Ayatana AppIndicator | link `wry`, `tao`, `tray-icon` |
| Runtime helpers | `zenity`, `xdg-utils` | file dialogs, "open in browser" |

All of them are installable from distro repos except Rust and Node, which you should get via **rustup** and **nvm** to avoid stale distro versions.

---

## 1. Install Rust (all distros)

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
rustc --version    # should be >= 1.75
