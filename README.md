# Opposing Force X

Porting **Half-Life: Opposing Force** to the **original Xbox**, on top of the
[Xash3D FWGS](https://github.com/FWGS/xash3d-fwgs) engine and
[hlsdk-portable](https://github.com/FWGS/hlsdk-portable).

This repository holds the **engineering log and the local patches**. It contains no
engine or game code of its own yet, and it contains **no game assets** — see
[Licensing](#licensing).

---

## Status

**Phase 1 — PC baseline: complete.**

| What | Result |
|---|---|
| Xash3D FWGS + Opposing Force, Linux **i386** (32-bit) | ✅ playable |
| Cross-compiled win32 gamedlls (mingw-w64 i686) + official win32 engine | ✅ runs on native Windows |
| Static gamedll linking (single binary, no `dlopen`) | ❌ abandoned — three separate blockers in the engine's `xshlib` mechanism |
| **Save/restore by offsets** (`XASH_ALLOW_SAVERESTORE_OFFSETS`) | ✅ **validated empirically** |

The save/restore validation is the important one: it is the mechanism a console port
needs when there is no symbol export table to resolve entity think/touch/use handlers
against. It was verified on the real Opposing Force campaign — a save written with
offset tokens, the engine closed, the save reloaded in a fresh process, with zero
pointer-resolution failures and a verified round-trip of the restored pointers.
Details in `BUILD.md` §15.

**Phase 2 — Xbox: toolchain working, game libraries building, engine blocked.**

| What | Result |
|---|---|
| [nxdk](https://github.com/XboxDev/nxdk) toolchain installed and verified | ✅ sample builds to a valid `.xbe` + ISO |
| [pbgl](https://github.com/fgsfdsfgs/pbgl) (OpenGL 1.x over pbkit) built | ✅ `libpbgl.lib`, 157 GL symbols |
| **Op4 game libraries cross-compiled with nxdk** | ✅ **231/231 sources, 0 unresolved symbols** — `opfor.dll` (1.59 MB) and `client.dll` (437 KB), PE32 i386, 660 + 71 exports |
| Fork [`maximqaxd/xash3d-fwgs_xbox`](https://github.com/maximqaxd/xash3d-fwgs_xbox) | ❌ does not configure — two files its own build references are absent from the repository |
| Loading those `.dll`s on the Xbox | ❌ **blocked** — nxdk has no PE loader |

The route is **nxdk**, and the game side of it works: the toolchain is clang targeting
`i386-pc-windows-msvc`, so the C++ ABI is MSVC's — the same one the original GoldSrc
gamedlls use. The game libraries compile with a small compatibility layer and no
modification to any hlsdk source file (`patches/hlsdk/hlsdk-nxdk-gamedll.patch`).

A useful side effect: the server library exports the 660 `Think`/`Touch`/`Use` handlers
that hlsdk marks with `EXPORT`, so `COM_FunctionFromName`/`NameForFunction` resolve by
name exactly as they do on Windows. That means this route needs neither Source-style
datamaps nor even the offset-based save/restore validated in Phase 1 — §15 becomes a
safety net rather than a requirement.

**The blocker is in the engine, not the game.** §16 chose nxdk partly on the claim that
it provides a working `LoadLibraryA`. That claim is wrong: nxdk's `LoadLibraryExA` is a
stub that always fails, and there is no PE loader anywhere in `nxdk/lib/`. The fork
compounds this by disabling Xash's own in-memory PE loader on Xbox and relying on that
stub. The likely way out is to re-enable Xash's `MemoryLoadLibrary` (464 lines, already
present) and supply the four Win32 calls nxdk lacks. Details and alternatives in §18.6–18.7.

The alternative base, [Half-LifeX](https://github.com/brentdc-nz/Half-LifeX), was
evaluated and rejected: it requires Visual Studio .NET 2003 plus Microsoft's
non-distributable Xbox XDK, is built on the older Xash3D 0.99 lineage, carries its own
11k-line OpenGL-over-D3D8 layer, and would require re-implementing save/restore as
hand-written Source-style datamaps across the game code. It remains an excellent
reference for the unsolved problem: fitting in 64 MB of RAM.

Full comparison in `BUILD.md` §16; Phase 2 work in §17 and §18.

---

## What's in here

```
BUILD.md                     the engineering log: every command, every failure, every fix
patches/engine/              local patches against xash3d-fwgs
patches/hlsdk/               local patches against hlsdk-portable (see LICENSE-NOTE.md)
LICENSE                      GPLv3
```

`BUILD.md` is written in Spanish and is deliberately verbose: it records the dead ends
as carefully as the successes, because most of the value of this kind of work is
knowing what does *not* work and why.

---

## Reproducing

Everything is in **`BUILD.md`**. In short:

1. §1 — enable i386 multiarch and install the 32-bit toolchain (Ubuntu/WSL).
2. §2–§3 — build the engine and the Opposing Force game libraries, 32-bit.
3. §6 — copy `valve/` and `gearbox/` from **your own legal copy** of the games.
4. §7 — launch with `-game gearbox`.
5. §11 — the local patches and how to reapply them after a `git pull`.

For the Xbox side, §17 covers installing and verifying nxdk and pbgl, and §18 covers
building the Op4 game libraries with it (`./waf configure --nxdk && ./waf build`).

The patches are not upstreamed. §11 explains what each one does and whether it is
mandatory or only needed for a specific experiment.

---

## Licensing

This repository is **dual-provenance on purpose**. Read this before reusing anything.

### The repository itself — GPLv3

`LICENSE` (GPLv3) covers this repository's own content: `README.md`, `BUILD.md`, and
the patches under **`patches/engine/`**, which are derived from
[Xash3D FWGS](https://github.com/FWGS/xash3d-fwgs) and therefore inherit its GPLv3.

### `patches/hlsdk/` — NOT GPLv3

The patches under **`patches/hlsdk/`** are derived from
[hlsdk-portable](https://github.com/FWGS/hlsdk-portable), which is a fork of the
**Half-Life SDK by Valve**. They are governed by that SDK's own license terms, **not**
by this repository's GPLv3. See
[`patches/hlsdk/LICENSE-NOTE.md`](patches/hlsdk/LICENSE-NOTE.md) and the
[upstream LICENSE](https://github.com/FWGS/hlsdk-portable/blob/master/LICENSE).

Putting the GPLv3 `LICENSE` at the repository root does **not** relicense those files,
and no such relicensing is claimed or intended.

### Game assets — not here, not ever

**This repository contains no game assets and never will.** No maps, models, textures,
sounds, WADs, PAKs, save files, `liblist.gam`/`gameinfo.txt` from a retail install, and
no compiled retail game libraries.

To run any of this you need **your own legally acquired copies** of:

- **Half-Life** (the `valve/` directory)
- **Half-Life: Opposing Force** (the `gearbox/` directory)

Both are available on Steam. `BUILD.md` §6 tells you where to put them.

---

## Acknowledgements

- [FWGS](https://github.com/FWGS) — Xash3D FWGS and hlsdk-portable
- [@maximqaxd](https://github.com/maximqaxd) — the in-progress Xbox port of Xash3D FWGS
- [@fgsfdsfgs](https://github.com/fgsfdsfgs) — [pbgl](https://github.com/fgsfdsfgs/pbgl),
  OpenGL 1.x for the Xbox over nxdk/pbkit
- [XboxDev](https://github.com/XboxDev) — nxdk
- [@brentdc-nz](https://github.com/brentdc-nz) — Half-LifeX, the reference for what
  running Half-Life on this hardware actually costs
