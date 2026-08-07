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

Both are described in detail in `BUILD.md` §11.1 and §11.4.

## Scope

These patches touch **build files only** (`CMakeLists.txt`, `wscript`). They contain no
Half-Life SDK source code, no game logic, and no game assets.
