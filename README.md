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

**Phase 2 — Xbox: toolchain decision made, implementation not started.**

The chosen route is **[nxdk](https://github.com/XboxDev/nxdk)**, starting from
[`maximqaxd/xash3d-fwgs_xbox`](https://github.com/maximqaxd/xash3d-fwgs_xbox) — an
in-progress port of modern Xash3D FWGS to the original Xbox that uses the same
codebase and the same waf build system as the PC work above. Notably, nxdk provides
`LoadLibraryA`, so game libraries load dynamically on Xbox and the static-linking dead
end does not apply.

The alternative, [Half-LifeX](https://github.com/brentdc-nz/Half-LifeX), was evaluated
and rejected as a base: it requires Visual Studio .NET 2003 plus Microsoft's
non-distributable Xbox XDK, is built on the older Xash3D 0.99 lineage, carries its own
11k-line OpenGL-over-D3D8 layer, and would require re-implementing save/restore as
hand-written Source-style datamaps across the game code. It remains an excellent
reference for the unsolved problem: fitting in 64 MB of RAM.

Full comparison in `BUILD.md` §16.

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
