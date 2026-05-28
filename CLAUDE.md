# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Operational Rules (mandatory)

1. **Two backups before touching any file.** Copy to a timestamped name in the same directory AND copy to a `garbage/` folder at repo root before any edit or overwrite. Example: `RSCTransfer.py` → backup as `RSCTransfer_bak_YYYYMMDD_HHMMSS.py` AND `garbage/RSCTransfer_bak_YYYYMMDD_HHMMSS.py`.
2. **Never delete anything.** Move unwanted files to `garbage/` at repo root. Do not `rm`, do not overwrite without backup.
3. **Every file must have a unique name.** Never reuse a backup filename. Include timestamp down to the second.
4. **No unauthorized changes.** If a task is unclear, ask before touching files. Do not "clean up" or refactor anything outside the stated scope.
5. **Never edit a pre-existing file directly.** Copy it to a new uniquely named file (date+time stamp in the name), edit the copy, never the original.
6. **Never push to main.** All commits stay on your working branch only. Pushing to `main` is forbidden unless the user gives explicit triple confirmation in the same conversation.

---

## Project Overview

This repo contains **RSCTransfer**, a Windows GUI application (Python + tkinter) that runs a local HTTPS MITM proxy to capture GTA Online character initialization payloads from one Rockstar Social Club account and inject them into a different account's session. The intended use case is migrating a GTA Online character from an Xbox account to a PC account after Rockstar discontinued official cross-platform transfers.

Current release: **v1.2.0** (adds Xbox console proxy support).

---

## Repository Layout

```
RSCTransfer_v1.1.0.zip    ← previous release (reference copy)
RSCTransfer_v1_2_0.zip    ← current release (source of truth)
READ                       ← full development chat transcript / history
```

The working source lives **inside the zip** at `rsctransfer/`:

| File | Purpose |
|---|---|
| `RSCTransfer.py` | Everything — cert gen, proxy engine, capture/inject logic, tkinter GUI (~1639 lines) |
| `test_proxy.py` | 18 functional tests (tkinter mocked out, tests run headlessly) |
| `build_exe.bat` | PyInstaller one-file build → `dist/RSCTransfer.exe` |
| `README.md` | User-facing docs |

---

## Running Tests

```cmd
cd rsctransfer
pip install cryptography
python test_proxy.py
```

Expected: `18 passed, 0 failed`. Tests mock tkinter at module level so no display is needed.

To run a single test, isolate the relevant block by line number — the test file is not a `unittest.TestCase` subclass, it's a linear script with `ok()`/`fail()` helpers.

---

## Building the .exe

```cmd
cd rsctransfer
pip install pyinstaller cryptography
build_exe.bat
```

Output: `dist\RSCTransfer.exe` — single-file, no Python runtime required.

---

## Architecture

### Proxy engine (single-file, threading-based)

- **`ProxyServer`** — `socketserver.ThreadingTCPServer` subclass. One thread per connection. Binds `0.0.0.0:8080` (LAN-accessible so Xbox can route through it).
- **`ProxyHandler`** — handles both plain HTTP and `CONNECT` tunnels. On CONNECT, performs TLS interception: generates a per-host cert signed by the local CA (`CertCache`), wraps the client socket in that cert, then opens a real TLS connection upstream.
- **`_forward_plain`** — serves `GET /ca.crt` (plain HTTP, no CONNECT) for Xbox cert download; all other plain requests are forwarded and optionally captured.
- **`_forward_ssl`** — intercepts HTTPS, parses response via `parse_http_response`, runs `_check_capture` on RSC domain responses, runs `_check_inject` if inject mode is on.

### Capture / Inject

- **`Capture`** dataclass — stores host, path, method, status, decoded body, content-type, timestamp. `is_char` is True if the lowercased body contains any of `CHAR_SIGNALS`.
- **`CaptureStore`** — thread-safe list + `threading.Lock`. GUI polls via a `queue.Queue`.
- **Inject mode** — when ON, the proxy matches incoming RSC requests against stored captures (exact `host+path` first, path-only fallback for region drift) and substitutes the saved body.

### Certificate management

- `generate_ca()` → self-signed root CA (10-year, RSA-2048)
- `generate_host_cert()` → per-hostname cert with SAN, SKI, AKI (required by Python 3.13+ / modern OpenSSL)
- `install_ca_windows()` → `certutil -user -addstore Root` (no admin)
- `CertCache` — LRU-like dict, one host cert per hostname, generated on first CONNECT

### GUI (tkinter, 4 tabs)

- **Setup** — CA generate/install/remove, proxy port, PC proxy instructions, Xbox mode toggle (LAN IP display, cert download URL, export-to-USB button, Xbox Settings walkthrough)
- **Capture** — start/stop proxy, live capture list, preview pane, save-to-file, send-to-inject
- **Inject** — inject mode toggle, queued payloads list, clear
- **Log** — scrolling proxy event log

### New in v1.2.0

- `get_local_ips()` — detects LAN IP via UDP connect trick + hostname resolution
- `GET /ca.crt` endpoint in `_forward_plain` — serves CA cert over plain HTTP so Xbox Edge can download and install it at `http://<LAN-IP>:<port>/ca.crt`
- Xbox Mode UI section — LAN IP label, cert URL, export-to-USB button, step-by-step Xbox proxy instructions
- `_refresh_lan_ip()`, `_do_export_ca_usb()` — supporting methods

---

## Key Constants

| Name | Value | Notes |
|---|---|---|
| `RSC_DOMAIN_SUFFIXES` | `rockstargames.com`, `.net`, `socialclub.*` | Strict suffix check (not substring) to prevent domain spoofing |
| `CHAR_SIGNALS` | 14 lowercase keywords | Triggers `is_char` flag on a capture |
| `DATA_DIR` | `%APPDATA%\RSCTransfer\` | Certs and captures stored here |
| `DEFAULT_PORT` | `8080` | Configurable in Setup tab |
