# HTTPLUA Server Pages (`.lp`) — Specification

**Project:** httplua · **Status:** Final · **Version:** 1.0

PHP-/JSP-style mixing of HTML and Lua via tags. A `.lp` file is transpiled into a
*single* Lua chunk in which literal text becomes writer calls and code between the tags
is taken verbatim. The model follows LuaPages/CGILua and is conceptually identical to
PHP/JSP/ERB.

---

## 1. Goal & Scope

Dynamic endpoints become writable as mixed HTML/Lua files
(`<h1>Hello, <?lua= name ?></h1>`) instead of emitting every byte through `print()`.

**In scope:** tag dialect, transpiler (template → Lua source), loader and routing in
HTTPLUA, output and content-type model, optional compiled-template cache.

**Out of scope / non-goals:**

- No changes to **lua370**. The engine stays a generic Lua 5.4 port; templating is
  web semantics and belongs in the CGI glue.
- No include/layout mechanism, no template inheritance (deferred to Phase 3).
- No replacement of the existing `.lua` script mode. Both modes coexist.
- **No state pooling** (reusing a `lua_State` across requests): leakage and sandbox
  risk. One fresh state per request.
- **No sessions / application state** (PHP `$_SESSION`): if ever required, a separate
  spec, UFS-backed, not in memory.

---

## 2. Tag Syntax

Two forms, deliberately using an explicit `lua` rather than a bare `<? ?>` to avoid
collision with `<?xml ... ?>` in XHTML:

| Tag             | Meaning                                    |
|-----------------|--------------------------------------------|
| `<?lua  ... ?>` | Statement block, **no** output             |
| `<?lua= ... ?>` | Expression, result is written to output    |
| (anything else) | Literal text, emitted verbatim             |

File extension: **`.lp`** (alternatively `.lhtml`). `.lua` remains script mode, unchanged.

Future extension (Phase 3): `<?lua== ... ?>` for *raw* unescaped output vs. `<?lua= ... ?>`
with HTML escaping as XSS protection. For now `<?lua=` emits raw.

---

## 3. Architecture

Everything lives in **HTTPLUA**. Three integration points.

### 3.1 Transpiler & loader in HTTPLUA, not in `lauxlib`

A dedicated function `httplua_template_load()` reads the file through the **public libufs
API** (`ufs_fopen` / `ufs_fclose` — the same path `readable()` already uses), transpiles
in memory, and hands the result to `lua_load()`. No `lauxlib` internals are touched;
HTML-template knowledge stays out of the generic auxiliary library. (The BOM/shebang skip
of `luaL_loadfilex` is irrelevant for `.lp`, since the file starts with HTML literal text,
not `#!`.)

### 3.2 Routing by extension

The router in `main_lua()` (`httplua.c`) selects the loader **purely by file extension**.
`.lua` continues through `luaL_dofile()` unchanged; `.lp`/`.lhtml` go through
`httplua_template_load()` + `lua_pcall()`. The existing `.lua` path is byte-for-byte
untouched.

```
Request → main_lua()
            │  switch on file extension only
            ├─ ".lp"/".lhtml" → httplua_template_load()  →  lua_pcall  →  buffer ─┐
            │                     (libufs read → transpile → lua_load)            │
            └─ ".lua"          → luaL_dofile()  (unchanged legacy path)           │
                                                                                  ▼
                                                  flush buffer → http_resp + http_printf → httpc
```

Server configuration directive: `MOD=HTTPLUA *.lp` (`CGI=` works as a deprecated alias
but emits `HTTPD410W`).

### 3.3 Why C, not Lua

The decisive factor is **bootstrap avoidance**, not caching. A Lua transpiler needs a
running `lua_State` itself — which means injecting module code into *every* short-lived
state, including on every cache *miss*, which is exactly when transpilation runs. A C
transpiler runs with no state and feeds `lua_load()` directly. That the later bytecode
cache can also only persist in C (§6) is a downstream benefit of the same root, not a
separate primary argument.

---

## 4. Transpilation Model

A template becomes a *single* Lua chunk: literal text → writer calls, code spans verbatim.
The writer is passed in as a **vararg local** (not via `_ENV`), so a later sandbox cannot
reach it.

**Input** (`hello.lp`):

```html
<?lua local name = http.vars.QUERY_STRING or "World" ?>
<h1>Hello, <?lua= name ?>!</h1>
<?lua for _, it in ipairs({"Hercules","MVS","Lua"}) do ?>
  <li><?lua= it ?></li>
<?lua end ?>
```

**Generated chunk** (conceptual):

```lua
local _out = ...                       -- writer passed in as vararg
local name = http.vars.QUERY_STRING or "World"
_out("\n<h1>Hello, ")
_out(tostring(name))
_out("!</h1>\n")
for _, it in ipairs({"Hercules","MVS","Lua"}) do
_out("\n  <li>")
_out(tostring(it))
_out("</li>\n")
end
_out("\n")
```

**Mapping rules:**

- Literal text `T` → `_out("<T encoded as a Lua string literal>")`
- `<?lua= e ?>` → expression `e` is **normalized to a string** and handed to the writer.
  The normalization step (in the PoC, `tostring`) is the natural hook for later HTML
  escaping or other writer filters.
- `<?lua s ?>` → `s` verbatim, on its own line.

**Literal escaping:** a small string-literal escaper produces valid Lua string literals.
The output buffer is a `luaL_Buffer` (self-growing). Templates are small → read the whole
file, then scan (no streaming parser required).

**Alternative without an escaper (Phase 2):** instead of emitting literals as source text,
place them in a table referenced by index (`_out(_lit[1])`) passed to `lua_load()` as an
upvalue. The bytes then never pass through the Lua lexer.

### 4.1 Parser: deliberately limited scanner in Phase 1

Phase 1 implements **only a simple scanner**. Active rule: **no `?>` inside Lua code.**
These cases confuse the naive scanner and are *not* supported in Phase 1:

```lua
print("?>")        -- string literal
local x = "?>"     -- string literal
-- ?>              -- line comment
--[[ ?> ]]         -- long-bracket comment (also [==[ ?> ]==], any depth)
```

Handling these correctly *is* already a small Lua lexer (string and long-bracket
tracking) — there is no sensible middle ground. It moves to Phase 3.

---

## 5. Output & Content-Type Model (buffered)

Template output is **buffered**, not streamed: `_out` writes into an in-memory
`luaL_Buffer`; the flush to `httpc` happens at the **end** of the request.

- **Clean error semantics.** If the Lua chunk fails mid-render, nothing has been sent yet
  — HTTPLUA can return a clean `500` instead of `200` plus half an HTML page (the classic
  PHP problem with already-flushed output). The existing script path also buffers (via the
  STDOUT dataset); the template path keeps that property, in memory rather than via a
  dataset.
- **Content-Type overridable until flush.** For `.lp`, `text/html` is the default but
  **committed lazily**: as long as nothing has been sent, the template may change the
  header (e.g. `http.header("Content-Type", "application/json")`). This lets JSON/XML
  endpoints be written as `.lp` too. The `<`-sniffing heuristic from
  `process_print`/`process_stdout` is not used on the template path — the content type is
  explicit, not guessed.

**Escape valve `http.flush()`** (analogous to `ob_flush`): for the rare case of very large
responses, an explicit flush deliberately commits the header and switches to streaming from
that point on. Default remains a single flush at end of request.

`STDERR` capture stays active; runtime errors are logged as before.

---

## 6. Cross-Request State

**Phase 1 requires nothing across request boundaries.** The `lua_State` (created →
closed), the output `luaL_Buffer`, and the STDOUT/STDERR temp datasets all live per
request. The feature is correct with zero persistent state.

The **only** cross-request state is the **bytecode/transpile cache of Phase 2** — and it
is purely a *performance* optimization, not a correctness requirement. This de-risks
Phase 1: anchor and concurrency questions only need answering once the cache is built.

### 6.1 Anchor: the existing `cgictx` registrar

httpd already provides a generic per-CGI context registrar, `http_cgictx_get()`
(introduced for mvsMF), reachable through the httpx vector. HTTPLUA obtains its context
block idempotently per request:

```c
HSPCTX *ctx = http_cgictx_get(httpd, "HTTPLUA", sizeof(HSPCTX));
if (!ctx) { /* array full / no core -> run without cache (= Phase-1 behavior) */ }
/* bytes 0-7 = eyecatcher (do not touch), rest zeroed on first create.            */
```

The eyecatcher is the 8-byte value `"HTTPLUA"` (7 characters plus the implicit NUL — the
8 bytes the registrar compares via `memcmp(..., 8)`).

Registrar properties (find-or-create per 8-byte eyecatcher): the block is allocated with
`__getm()` in **subpool 0** (address-space-wide, `SZERO=YES` on the worker ATTACH), so it
outlives the LINK return that started the CGI and the worker shrink, and lives until the
address space ends. Same eyecatcher → same pointer; full array → `NULL`, no ABEND. The
persistence guarantee is thus **explicitly** bound to the address-space lifetime, with no
ABI change and no reclaim of a reserved struct slot. `HSPCTX` holds the cache root
(hash/list head) plus a latch word.

### 6.2 Two pitfalls the mechanism forces

- **Cache entries also need SP0 lifetime.** Only the *block* comes from `__getm()`. What
  HTTPLUA stores *into* it — bytecode buffers, hash nodes — must **not** be `malloc()`:
  the CGI's malloc storage is reclaimed at request teardown (`@@exit`), and the block
  would then point at freed storage. Allocate entries through the same SP0 `__getm()`
  path.
- **Mutation inside the block is HTTPLUA's own responsibility.** The registrar lock
  (CLIBLOCK ENQ, STEP scope) covers only the block's find-or-create. Insert/replace of
  cache entries across concurrent workers needs HTTPLUA's *own* serialization — the
  `lock()`/`unlock()` primitives in `httpgctx.c` are httpd-internal and not exported to
  modules. Since entries are immutable once built, a **latch (a compare-and-swap word in
  `HSPCTX`, or a private ENQ) around insert/replace only, with lock-free reads** is
  sufficient — the same pattern as UFS370 Konzept#2/#3.

### 6.3 Graceful degradation & configuration

If `http_cgictx_get()` returns `NULL` (array full per `CGI_CONTEXT_POINTERS`, or no core),
HTTPLUA continues without a cache — Phase 2 never breaks the feature, it only loses the
optimization. Requirement: `CGI_CONTEXT_POINTERS` must be large enough for all
context-using CGIs (mvsMF + HTTPLUA + …).

---

## 7. EBCDIC

Non-critical. The delimiters `< ? = >` are all in the **invariant** character set. The
whole pipeline stays EBCDIC: the transpiler scans EBCDIC bytes and emits EBCDIC Lua
source; `lua_load()` compiles EBCDIC source (lua370 already does this for `.lua`); the
EBCDIC→ASCII conversion happens centrally on the wire in httpd/`httpxlat` — **not** in the
template.

---

## 8. Caching (Phase 2)

On Hercules it pays to transpile + compile each file exactly once.

- **Key:** UFS path + `mtime`. **Value:** dumped bytecode (`lua_dump` → `luaL_loadbufferx`
  on the next request).
- **Anchor & concurrency:** see §6 (`http_cgictx_get`, eyecatcher `"HTTPLUA"`, SP0
  lifetime; latch around insert/replace, lock-free reads).
- **Entry storage:** bytecode buffers and hash nodes via `__getm()` (SP0), not `malloc()`
  — otherwise reclaimed at request teardown (§6.2, pitfall 1).
- **Invalidation:** re-transpile on `mtime` change. Because `__getm` blocks are never
  freed individually, a replace *leaks* the old buffer — bounded by the number of template
  edits during server uptime, acceptable on Hercules; add a buffer free-list if it ever
  matters.
- **Version safety.** The cache lives in the address space and vanishes with the httpd STC;
  a new HTTPLUA/lua370 build (restart) therefore invalidates it implicitly. The point only
  becomes sharp if the cache ever goes **persistent** (UFS/dataset) — then it needs an
  explicit version/format stamp (lua370 version + bytecode format). As a backstop,
  `luaU_undump` already checks the bytecode header (signature, version, format, int/float
  sizes) and rejects incompatible bytecode on load.

---

## 9. Sandbox & Error Handling

**Sandbox (Phase 3):** in production, set `_ENV` to a curated whitelist, not full `_G`.
The `_out` vararg local is unaffected.

**Error line numbers.** Lua reports errors relative to the *generated* chunk, not the
`.lp` source. Approach: first **preserve as many source newlines as possible in the
generated Lua code** (emit literal blocks so that consumed `\n` land as real line breaks
in the chunk), so line numbers roughly track — a fraction of the effort of a real source
map and good enough for everyday use. A real map only if needed.

**Parser limits:** see §4.1 (no `?>` in Lua code in Phase 1).

---

## 10. Phases

**Phase 1 — PoC.** C transpiler + `httplua_template_load()` + extension router; buffered
output; `text/html` default committed lazily; simple string-literal escaper; `.lp`
extension; **no** cross-request state. *Goal:* `hello.lp` renders correctly end-to-end,
including a clean `500` on render error.

**Phase 2 — Performance & diagnostics.** Bytecode cache (`cgictx` anchor, latch); source-
newline preservation for errors; newline-after-`?>` handling; optional literal table
instead of the escaper.

**Phase 3 — Hardening & convenience.** `_ENV` whitelist; real tokenizer (`?>` inside
strings/comments/long brackets); escaped-vs-raw output (`<?lua=` / `<?lua==`);
include/layout mechanism.

---

## 11. Design Decisions

**D1 — Output buffered, not streamed.**
*Rationale:* clean `500` on mid-render error; content type overridable until flush.
*Consequence:* `luaL_Buffer` per request, flush at end; `http.flush()` as an optional
streaming valve. Templates are small → memory cost is negligible.

**D2 — Transpiler/loader in HTTPLUA, not in `lauxlib`.**
*Rationale:* HTML-template knowledge does not belong in the generic Lua auxiliary library;
access goes through the public libufs API anyway. *Consequence:* `httplua_template_load()`;
`lauxlib` internals untouched.

**D3 — Transpiler in C.**
*Rationale:* bootstrap avoidance (no module injection into every short-lived state, not
even on a cache miss); locality to the file loader; cache persistence as a follow-on.
*Consequence:* a pure C path up to `lua_load()`.

**D4 — Only cross-request state = the Phase-2 cache, anchored via the `cgictx` registrar.**
*Rationale:* correct, explicit address-space lifetime via the existing `http_cgictx_get()`
mechanism (as used by mvsMF), with no ABI change and no reserved-slot reclaim.
*Consequence:* `HSPCTX` keyed by the 8-byte eyecatcher `"HTTPLUA"`; cache entries also SP0
`__getm()` (not malloc); own latch around insert/replace because of the worker pool;
graceful degradation to "no cache" on `NULL`; no state pooling, no sessions.

---

## 12. Open Points

1. **Extension final:** `.lp` vs. `.lhtml` for the `MOD=HTTPLUA *.lp` routing entry.
2. **Config sizing:** `CGI_CONTEXT_POINTERS` must cover all context-using CGIs
   (mvsMF + HTTPLUA + …). (Phase 2.)
3. **XSS default:** should `<?lua=` eventually escape rather than emit raw? Migration path
   if existing templates expect raw.
4. **Include/layout:** is there demand? If so, sharpen the Phase-3 scope (e.g.
   `http.include("header.lp")` reusing the same cache layer).

---

## Revision History

- **1.0** — Final. Transpiler/loader in HTTPLUA via public libufs; routing by extension
  (`MOD=HTTPLUA *.lp`); C transpiler (bootstrap avoidance); buffered output with lazy
  content-type and `http.flush()` valve; cross-request cache anchored via `http_cgictx_get`
  (eyecatcher `"HTTPLUA"`, SP0 lifetime, own mutation latch, graceful degradation);
  EBCDIC end-to-end; Phase-1 PoC carries no cross-request state.
