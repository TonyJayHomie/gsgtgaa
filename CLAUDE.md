# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository owner's operating rules (read first)

These are hard constraints set by the repo owner. They take priority over default behavior:

- **Never push to `main`, and never push anywhere without explicit triple approval.** Keep every commit local to the working branch (`claude/…`) only. Do not upload, push, or open a PR to any remote — *not even the feature branch* — unless the owner has explicitly confirmed that specific push **three times**. The `main` branch is off-limits regardless of approval. Committing locally is fine and encouraged; publishing is what requires the triple sign-off.
- **Never delete anything.** No `rm`, no destructive overwrite — not even of an empty or junk file. To get something out of the way, create a `GARBAGE/` folder at the repo root and move it there.
- **Two backups before touching any existing file.** Before editing or overwriting a file, copy it to two uniquely-named backups first (e.g. `RSCTransfer.py.20260528-1.bak`, `RSCTransfer.py.20260528-2.bak`), then make the change.
- **Never edit a pre-existing file in place — work on a copy.** Do not alter, rewrite, or overwrite any file that already exists. Copy it to a new, uniquely-named file (preferably with a date+timestamp in the name, e.g. `RSCTransfer.20260528-2041.py`) and make all changes in that copy; the original must stay byte-for-byte untouched. The sole exception is an explicit, one-time permission from the owner for a specific file — and even then, take two backups before **and** two after the edit (four total).
- **Unique filenames always.** Never reuse a name in a way that clobbers an existing file. New artifacts get their own distinct name (version or timestamp suffix).
- **No unauthorized changes.** Change only what was explicitly requested. Do not reformat, "tidy up," or edit unrelated files or lines.
- **History/rationale lives in `READ`.** `READ` is the full exported Claude.ai transcript that produced this tool, and each README's changelog explains every fix. When unsure *why* something is the way it is, read those instead of guessing.

`GARBAGE/` and backup files should not be committed unless explicitly asked.

## What this repo is

A single tool, **RSCTransfer** — a Windows desktop app (Python + `tkinter`) that runs a local HTTPS man-in-the-middle proxy to capture a GTA Online character/account payload from one Rockstar Social Club account and replay ("inject") it into another account's session. Designed to be paired with FSL (`WINMM.dll`) so injected state persists locally instead of being overwritten by Rockstar's next server-side save.

The repo does **not** contain an extracted source tree. Source ships as versioned zips at the repo root:

| File | Version | Notes |
|---|---|---|
| `files (29).zip` | 1.0.0 | Original draft. Pre-bugfix — **fails the test suite** (no `_utcnow`, uses deprecated `datetime.utcnow()`, host certs lack an AKI so live TLS interception fails). |
| `RSCTransfer_v1.1.0.zip` | 1.1.0 | Bugfix release. Passes 18/18. |
| `RSCTransfer_v1_2_0.zip` | 1.2.0 | **Latest / canonical.** Adds Xbox-console proxy support (LAN-IP display, `GET /ca.crt` endpoint for Xbox Edge cert install, export-to-USB). Passes 18/18. |

Each zip contains a `rsctransfer/` dir with `RSCTransfer.py`, `test_proxy.py`, `build_exe.bat`, `README.md`. The code's `APP_VER` constant is the source of truth for version — note the v1.2.0 `README.md` title still reads "Version 1.1.0" (stale text; the code is 1.2.0).

## Commands

Work from an extracted copy of the latest zip; extract to a scratch dir so the committed zips are never overwritten:

```bash
unzip -o "RSCTransfer_v1_2_0.zip" -d /tmp/rsc
cd /tmp/rsc/rsctransfer
```

- **Install deps:** `pip install cryptography` (the only runtime dependency). Optional: `brotli` (Brotli response decoding degrades gracefully when absent).
  - Gotcha seen in this sandbox: if importing `cryptography` raises `ModuleNotFoundError: No module named '_cffi_backend'`, its CFFI backend is missing — run `pip install cffi`.
- **Run the app:** `python RSCTransfer.py` (opens the `tkinter` GUI; Windows is the intended target).
- **Run tests:** from the directory containing `RSCTransfer.py`, run `python test_proxy.py` → expect `18 passed, 0 failed`. The suite mocks `tkinter`, so no display is needed, and it includes a real localhost TLS MITM + inject round-trip (Tests 11–12). `test_proxy.py` is byte-identical in 1.1.0 and 1.2.0.
  - There is no single-test runner — it is one flat script. To run a single check, comment out the others or copy the block.
- **Build the .exe (Windows only):** `build_exe.bat` → PyInstaller one-file build at `dist\RSCTransfer.exe`.

Verified in this environment (Linux, Python 3.11, cryptography 41): 1.1.0 and 1.2.0 both pass 18/18; the 1.0.0 draft errors immediately.

## Architecture (everything is in the single `RSCTransfer.py`)

Layered, bottom to top:

1. **Certificates** — `generate_ca()` makes a self-signed 10-year root (with a SubjectKeyIdentifier). `generate_host_cert()` makes a per-host leaf signed by that CA carrying SAN + SKI + an AuthorityKeyIdentifier derived from the CA's SKI — the AKI is mandatory for Python 3.13 / modern OpenSSL to accept the chain. `CertCache` is a thread-safe per-host leaf cache. `install_ca_windows()` / `remove_ca_windows()` wrap `certutil -user -addstore/-delstore Root` (user store, no admin needed).
2. **HTTP parsing** — `parse_http_response()` + `_dechunk()` + `decode_body()` handle status/headers, chunked transfer-encoding (including chunk extensions), and gzip/deflate/brotli. `read_full_response()` / `read_full_request()` read a complete message using Content-Length first, then a chunked terminator, then connection-close as fallbacks.
3. **`Capture`** — one intercepted response. `CHAR_SIGNALS` is the substring heuristic (`characterdata`, `wallet`, `money`, …) that sets `is_char` (the ★ flag). Serializes binary-safe via hex (`to_dict` / `from_dict`).
4. **`MITMProxy`** — threaded socket server (binds `0.0.0.0:<port>`, default 8080). Per connection: on `CONNECT`, if the host matches a Rockstar suffix (`_is_rsc` — **strict** suffix match, so `rockstargames.com.evil.tld` and `evil-rockstargames.com` are rejected) it terminates TLS with a generated leaf cert and runs `_intercept_loop`; otherwise it blind-`_tunnel`s the bytes. `_intercept_loop` either (a) in **inject mode** replaces the response for a matching `host+path` from `inject_map` (with a trailing-path fallback for regional endpoint differences), or (b) forwards upstream and enqueues a `Capture`. Plain-HTTP `GET /ca.crt` serves the CA for Xbox Edge install. ALPN is pinned to `http/1.1` on both client and upstream sides. `set_inject()` is lock-guarded.
5. **`RSCTransferApp(tk.Tk)`** — four tabs: **Setup** (generate/install CA, port, PC-vs-Xbox proxy instructions, LAN-IP + `ca.crt` URL), **Capture** (start/stop proxy, capture list with ★ filter, JSON preview, "Send selected to Inject"), **Inject** (queue keyed by `host+path`, "Inject mode ON" toggle), **Log**. The proxy runs on a background thread and communicates with the UI through two `queue.Queue`s drained by `_poll_queues()` on the Tk event loop every 150 ms.

**End-to-end flow:** install CA + start proxy → point the device's system proxy at it → load GTA on account A (captures ★ payloads) → "Send selected to Inject" → switch to account B, enable inject, load GTA (proxy serves A's payload). FSL must run alongside or the next server save overwrites the injected state.

**Platform note:** Windows-only calls (`certutil`, `ms-settings:`, `os.startfile`) are guarded by `os.name == "nt"`. The cert / HTTP / proxy / capture core — and the entire test suite — are cross-platform.

## Known limits (from the README; useful when debugging)

- **Cert pinning** — pinned Rockstar endpoints cannot be intercepted; they appear as `upstream TLS fail` in the Log and yield no ★ capture. Not fixable client-side.
- **Region mismatch** — NA/EU/AS use different endpoints; capture on the same region you intend to inject on. The trailing-path fallback covers most, not all, cases.
- **Persistence** — injects are client-side only; without FSL running, Rockstar's next save overwrites them.
