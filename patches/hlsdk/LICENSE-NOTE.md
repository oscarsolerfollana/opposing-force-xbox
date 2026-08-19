# Licensing note for `patches/hlsdk/`

**The patches in this directory are NOT covered by the GPLv3 `LICENSE` at the root of
this repository.**

They are diffs against
[FWGS/hlsdk-portable](https://github.com/FWGS/hlsdk-portable), which is a fork of the
**Half-Life SDK by Valve Corporation**. Anything derived from that SDK — including
these patches — is governed by the SDK's own license terms.

- Upstream license: <https://github.com/FWGS/hlsdk-portable/blob/master/LICENSE>

Placing a GPLv3 `LICENSE` file at the repository root does not relicense these files,
and no such relicensing is claimed or intended. If you reuse them, you are bound by
Valve's SDK terms, not by the GPLv3.

## What these patches are

| File | Target | What it does |
|---|---|---|
| `mingw-server-snprintf.patch` | `dlls/CMakeLists.txt` | Guards `-D_snprintf=snprintf` with `if(NOT MINGW)`, mirroring a guard that already exists in `cl_dll/CMakeLists.txt`. Without it the server library does not cross-compile with mingw-w64. |
| `hlsdk-vcs-info-inline.patch` | `dlls/wscript`, `cl_dll/wscript` | Compiles `game_shared/vcs_info.c` directly into the server and client targets instead of linking it as a separate static library, making the game libraries self-contained. |
| `hlsdk-nxdk-gamedll.patch` | `wscript`, `dlls/wscript`, `cl_dll/wscript`, `scripts/waifulib/xcompile.py`, `external/nxdk/*` (new) | Adds an `--nxdk` cross-compilation target for the original Xbox, following the same pattern hlsdk already uses for Nintendo Switch and PS Vita. Builds `opfor.dll` + `client.dll` as PE32 i386 with the nxdk toolchain. |
| `hlsdk-hl-nxdk.patch` | same files, `master` branch | The same `--nxdk` target on hlsdk's **`master`** branch (plain Half-Life) instead of `opfor`. Half-Life base was used to validate the whole console pipeline before switching to Opposing Force, because its assets were already on the console and known good. Builds `hl.dll` + `client.dll`. |

The first two are described in detail in `BUILD.md` §11.1 and §11.4; the nxdk patches in
§18 and §24.

> **`hlsdk-nxdk-gamedll.patch` is cumulative and already contains
> `hlsdk-vcs-info-inline.patch`.** Apply one or the other, not both.

> **`hlsdk-hl-nxdk.patch` and `hlsdk-nxdk-gamedll.patch` are the same work on two
> different branches.** They are not meant to be applied to the same tree: pick the
> branch you want to build. §24.0 explains why both exist and how the two are kept as
> git worktrees sharing one object store.

Both nxdk patches also carry the target's `-march=pentium3 -mno-sse2`, which is not an
optimisation preference: hlsdk's own `compiler_optimizations.py` appends
`-march=pentium-m` for every 32-bit x86 target, and a Pentium M has SSE2 while the
console's Pentium III does not. Without it the game libraries emit instructions the
hardware cannot execute (`BUILD.md` §24.7).

## Scope

`mingw-server-snprintf.patch` and `hlsdk-vcs-info-inline.patch` touch **build files
only** (`CMakeLists.txt`, `wscript`).

`hlsdk-nxdk-gamedll.patch` also touches build files (`wscript`, `xcompile.py`), plus it
**adds four new files under `external/nxdk/`**: a compatibility layer that supplies what
the nxdk libc (pdclib) does not provide — `va_list`, `<sys/types.h>`, `<memory.h>`, the
`DLL_PROCESS_*` constants, `POINT`/`*CursorPos`, and a small runtime with global
`operator new`/`delete` and `atof()`. That layer is original work, written for this
project and force-included with `-include`; it is not derived from the Half-Life SDK and
modifies no hlsdk source file. It is included in the patch so the patch is
self-contained.

None of these patches contain Half-Life SDK source code, game logic, or game assets.
