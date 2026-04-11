# CLAUDE.md — httplua

## Project Overview

HTTPLUA is the Lua CGI handler for the HTTPD web server on MVS 3.8j. It was
extracted from the HTTPD codebase (issue mvslovers/httpd#61) into a standalone
project to decouple the Lua runtime dependency from the HTTP server core.

HTTPLUA produces two load modules:
- **HTTPLUAX** — Lua function vector (no CRT, custom entry point)
- **HTTPLUA** — Lua CGI handler (standard CRT)

Both are registered as external CGI modules in HTTPD's Parmlib configuration:
```
CGI HTTPLUA /lua/*
```

## C Standard

This project uses `-std=gnu99` (same as HTTPD).

## Dependencies

- **crent370** — C runtime
- **lua370** — Lua interpreter
- **httpd** — for httpcgi.h (CGI module interface)
- **ufsd** — UFS filesystem access

## Configuration

Lua CGI configuration is via environment variables (set in DD:SYSENV or DD:ENVIRON):
- `CGILUA_DATASET` — PDS dataset for Lua scripts
- `CGILUA_PATH` — Lua package.path
- `CGILUA_CPATH` — Lua package.cpath
