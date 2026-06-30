# CLAUDE.md — httplua

## Project Overview

HTTPLUA is the Lua CGI handler for the **httpd** web server on MVS 3.8j. It was
extracted from the httpd codebase (mvslovers/httpd#61) to decouple the Lua
runtime from the HTTP server core. It embeds **lua370** (statically linked) and
runs Lua scripts from the UFS in response to HTTP requests.

It builds a single load module, **HTTPLUA**. (The earlier HTTPLUAX function
vector was removed in #2 — lua370 is now linked directly.)

## C Standard

`-std=gnu99` (same as httpd) for the CGI glue; the Lua sources are C89. The
build defines `LUA_USE_C89`, `LUA_USE_JUMPTABLE=0` and `LUA_USE_CTYPE=1` to match
lua370's ABI and its EBCDIC-correct lexer (see **Encoding**).

## Build (mbt v2)

httplua uses [mbt](https://github.com/mvslovers/mbt) v2 (cc370 host build).
`make` runs entirely on the host; MVS is only touched by `make deploy`.

```bash
make deps     # resolve + stage lua370, httpd, ufsd into .mbt/deps
make          # cross-compile + link the HTTPLUA load module -> build/HTTPLUA
make deploy   # XMIT + upload + RECEIVE into IBMUSER.HTTPLUA.V1R0M0D.LINKLIB
```

### Dependencies (project.toml)

| Dependency | Purpose |
|------------|---------|
| `mvslovers/lua370` | Lua 5.4 engine (statically linked into HTTPLUA) |
| `mvslovers/httpd`  | CGI module interface (`httpcgi.h`, libhttpd) |
| `mvslovers/ufsd`   | UFS filesystem access (libufs) |

**libc370** is the cc370 sysroot (`-lc`; provides the C runtime, `racf.h`,
`acee.h`), not a declared dependency. To develop against an unreleased lua370,
use a gitignored `.mbt/deps.local.toml` `[override]` pointing at `../lua370`.

## Source Files

- **httplua.c** — the CGI handler (`@@CRT0` entry via cgistart): resolves the
  script, sets up the Lua state, runs the script, streams stdout/stderr back.
- **cgihttpc.c / cgihttpd.c** — glue returning the httpd `HTTPC`/`HTTPD` from
  the CRT app slots.
- **lauxlib.c / liolib.c / loadlib.c** — CGI-patched copies of the matching Lua
  sources. They are linked directly, so they **override** lua370's archived
  versions (ld370 autocall only pulls *undefined* symbols from the archive).

## Configuration (httpd parmlib + UFS)

httpd is **UFS-only** — scripts live in the UFS (served by ufsd), not in MVS
datasets. The script path is configured in the **httpd parmlib member**
(`DD:HTTPPRM`), not in httplua:

```
UFS=1
DOCROOT=/www            # UFS root for scripts (UFSD must be running, path must exist)
MOD=HTTPLUA *.lua       # extension routing
```

With an **extension** pattern (`*.lua`), httpd sets `SCRIPT_FILENAME =
DOCROOT + request-path` and httplua opens that UFS file directly — e.g.
`GET /lua/app.lua` → `SCRIPT_FILENAME=/www/lua/app.lua`.

> A **prefix** pattern (`MOD=HTTPLUA /lua/*`) does NOT set `SCRIPT_FILENAME`;
> httplua then falls back to a legacy `CGILUA_PATH` lookup (a `;`-separated list
> of `?`-templates loaded from `DD:SYSENV`). Prefer extension routing.

Optional env (via `DD:SYSENV`/`DD:ENVIRON`, read by `loadenv`): `CGILUA_PATH`,
`CGILUA_CPATH` (Lua `package.path`/`cpath`). The old `CGILUA_DATASET` is no
longer read by the code.

## Encoding (EBCDIC — CRITICAL)

Lua scripts in the UFS are **EBCDIC** (authored with MVS tools). lua370 is built
with `LUA_USE_CTYPE=1` so its lexer uses libc370's EBCDIC-aware `<ctype.h>` —
otherwise `lctype.h`'s `#if 'A' == 65` auto-detect (mis-evaluated as host ASCII
by the cc370 cross-preprocessor) selects an ASCII ctype table and mis-lexes
EBCDIC source (e.g. `print`, `p` = 0x97, fails as "unexpected symbol"). httplua
sets the same flag for consistency. Script output is EBCDIC; httpd translates it
to the response codepage.

## Deploy / activation

`make deploy` only writes the deploy LINKLIB (`IBMUSER.HTTPLUA.V1R0M0D.LINKLIB`).
To activate under a running httpd:
1. IEBCOPY the `HTTPLUA` member into httpd's load library (e.g. `HTTPD.LINKLIBT`).
2. `MOD=HTTPLUA *.lua` (+ `DOCROOT`) in the httpd parmlib.
3. **Restart httpd** — `MOD=` modules are `__load()`ed at httpd startup, so a
   restart (not a per-request reload) picks up a new HTTPLUA and parmlib changes.

## Workflow

GitHub Issue + feature branch + PR for non-trivial changes. **Never reference
AI or Claude** in commits, comments, PRs, or anywhere in the project.
