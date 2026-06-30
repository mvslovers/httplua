# httplua

**httplua** is the Lua CGI handler for the [httpd](https://github.com/mvslovers/httpd)
web server on classic **MVS 3.8j**. It embeds the
[lua370](https://github.com/mvslovers/lua370) Lua 5.4 engine and runs Lua scripts
from the UFS in response to HTTP requests — so you can write dynamic web
endpoints in Lua on a Hercules-emulated MVS system.

It was extracted from httpd (mvslovers/httpd#61) into a standalone project and
builds a single load module, **HTTPLUA**.

## Features

- Lua 5.4 scripting for HTTP requests (via the embedded lua370 engine)
- Scripts served from the **UFS** (ufsd), addressed by path
- EBCDIC-aware — Lua source and output are EBCDIC; httpd handles the wire codepage

## Building

httplua uses [mbt](https://github.com/mvslovers/mbt) v2 — a host build with the
**cc370** toolchain. MVS is only touched by `make deploy`.

### Prerequisites

- The **cc370** host toolchain (it also provides the **libc370** sysroot)
- **Python 3.12+**
- An MVS 3.8j system reachable over IP (for `make deploy`)

### Build

```bash
git clone --recursive https://github.com/mvslovers/httplua.git
cd httplua
cp .env.example .env     # MVS connection for make deploy
make deps                # stage lua370, httpd, ufsd
make                     # build/HTTPLUA (on the host)
make deploy              # -> IBMUSER.HTTPLUA.V1R0M0D.LINKLIB
```

Dependencies (`project.toml`): `lua370` (the Lua engine, statically linked),
`httpd` (CGI interface), `ufsd` (UFS access). `libc370` is the cc370 sysroot.

## Installing / activating

1. Copy the `HTTPLUA` member from the deploy LINKLIB into httpd's load library
   (e.g. `HTTPD.LINKLIBT`).
2. Configure routing + the script root in the httpd parmlib member (`DD:HTTPPRM`):
   ```
   UFS=1
   DOCROOT=/www
   MOD=HTTPLUA *.lua
   ```
3. Restart httpd (it loads `MOD=` modules at startup).

A script at UFS `/www/hello.lua`:

```lua
print("Hello, World!")
```

is then reachable at `http://host:port/hello.lua`.

## Acknowledgments

Built on **httpd** and the **lua370** Lua port (originally by Michael Dean
Rayborn) for the MVS 3.8j community. Lua is by Roberto Ierusalimschy, Waldemar
Celes, and Luiz Henrique de Figueiredo (PUC-Rio).

## License

[MIT License](LICENSE)
