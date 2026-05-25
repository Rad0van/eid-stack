# eid-stack

A single-file Python 3 tool (stdlib only) that installs, updates, and removes the
Slovak **DITEC eID signing stack** on **Apple Silicon running [Asahi Linux](https://asahilinux.org/)**.

The eID software (DITEC D.Launcher 2 / D.Bridge 2, eID Klient, D.Signer Java, and
Disig WebSigner) is **x86-64 only** and assumes an ordinary x86 Linux with 4 KiB
memory pages. On aarch64 Asahi the kernel uses **16 KiB pages**, which the x86
binaries (jemalloc) can't tolerate, so they have to run inside a **[muvm](https://github.com/AsahiLinux/muvm)
microVM** (a 4 KiB-page guest, x86 emulated via **FEX**). Wiring that up by hand —
persistent VM, native-messaging wrappers, a pcscd bridge, an IPv6 loopback shim,
JRE placement — is a lot of fiddly steps. `eid-stack` reproduces the whole thing
with one command.

> Personal tooling for one specific machine (aarch64 Asahi + KDE Wayland, x86 via
> muvm/FEX). It is not a general-purpose installer and hard-codes assumptions about
> that environment.

## Components

| Component | What it does |
|---|---|
| `muvm-service` | Persistent `muvm` microVM as a systemd **user** service, the `~/.local/bin/muvm-guest` PATH symlink, slim wrappers, passt bound IPv4-only, and the `eid-ipv6-shim.service` (so browsers' `localhost`→`::1` reaches eID Klient). |
| `dlauncher` | D.Launcher 2 + the `dBridge2Nm` native-messaging host and the Brave/Chrome NM manifest. |
| `eid-klient` | Slovak **eID Klient** middleware (installs to `/usr/lib/eID_klient/`; **needs sudo**). |
| `dsigner` | Pre-fetches **D.Signer Java** + **Zulu JRE 21** into `~/.ditec/dlauncher2/products/` (exec bits preserved). |
| `websigner` | **Disig WebSigner** — fetches/extracts its `amd64` `.deb` (no arm64 build exists and its own updater can't run here) and re-points it through muvm: tray autostart, menu entry, browser native-messaging hosts. **Needs sudo.** |

## Requirements

- **Asahi Linux on Apple Silicon** (aarch64).
- **Python 3** (standard library only — no pip packages).
- **`muvm`** in `PATH` (a custom build with pcscd/loopback/overcommit patches is
  recommended; `muvm-service` symlinks `~/.local/bin/muvm-guest → ~/.cargo/bin/muvm-guest`).
- **`socat`** — for the IPv6 loopback shim (skipped with a warning if absent).
- **`sudo`** — required by `eid-klient` and `websigner` (run those from a real terminal).
- **`dpkg-deb`** — for `websigner`.

## Usage

```
eid-stack <command> [components...]

Commands:
  install   [comp...]   install component(s)            (default: all)
  update    [comp...]   re-download if upstream changed  (default: all installed)
  reinstall [comp...]   uninstall then install
  uninstall [comp...]   remove component(s)              (default: all installed)
  list                  show installed versions + update availability
  help                  show help

Components: muvm-service  dlauncher  eid-klient  dsigner  websigner   (or "all")
```

Components install/uninstall in dependency order regardless of the order given.

### Examples

```sh
eid-stack install                 # bootstrap the whole stack on a fresh box
eid-stack list                    # what's installed, versions, upstream drift
eid-stack update                  # re-fetch anything whose upstream changed
eid-stack reinstall eid-klient    # re-run one component (needs sudo)
eid-stack update websigner        # fetch+extract the latest .deb, then re-point
```

`update` uses HTTP `HEAD` (ETag / Last-Modified) to detect upstream changes.
`muvm-service` has no upstream and just re-applies its generated units/wrappers.

## State & layout

- **State:** `~/.config/eid-stack/state.json` — per-download ETag/Last-Modified,
  install timestamps, captured versions.
- **Generated wrappers:** `~/bin/dBridge2Nm.sh`, `~/bin/eID_Client.sh`,
  `~/bin/WebSigner.sh`, `~/bin/WebSignerTray.sh`.
- **systemd user units:** `dlauncher2-muvm.service`, `eid-ipv6-shim.service`.
- **Binaries:** D.Launcher under `~/.local/share/dlauncher2/`; eID Klient under
  `/usr/lib/eID_klient/`; D.Signer + JRE under `~/.ditec/dlauncher2/products/`;
  WebSigner under `/opt/disig/websigner/`.

## Environment-specific fixes baked in

These are non-obvious workarounds for the muvm/FEX environment that `eid-stack` applies:

- **IPv6 `localhost`** — Brave/Chromium resolve `localhost` to `::1` first, but
  passt forwards IPv4 only. `muvm-service` binds passt to `127.0.0.1` and installs
  `eid-ipv6-shim.service` (`socat ::1:15480 → 127.0.0.1:15480`); otherwise portals
  report *"eID klient is not running"*.
- **JRE executable bits** — `zipfile.extractall()` drops the modes stored in the
  Zulu JRE zip, leaving `bin/java` non-executable → D.Signer fails with
  `execve: Permission denied`. `dsigner` extracts with a mode-preserving helper.

## License / disclaimer

Personal automation, provided as-is with no warranty. It downloads the official
DITEC and Disig binaries from `slovensko.sk` / `eidas.minv.sk` / Disig's CDN and
verifies SHA-256 where the manifests provide it; those components remain under
their respective vendors' licenses.
