# Opposing Force X

Porting **Half-Life: Opposing Force** to the **original Xbox**, on top of the
[Xash3D FWGS](https://github.com/FWGS/xash3d-fwgs) engine and
[hlsdk-portable](https://github.com/FWGS/hlsdk-portable).

This repository holds the **engineering log and the local patches**. It contains no
engine or game code of its own, and it contains **no game assets** — see
[Licensing](#licensing).

---

## Status

**It runs.** Opposing Force boots, plays and is being debugged on real hardware: a
retail original Xbox, 128 MB, with the game libraries linked statically inside the
`.xbe`.

| What | Result |
|---|---|
| Half-Life (base game) on the console | ✅ playable — menu, maps, NPCs, weapons, decals, sound, controller |
| **Opposing Force** on the console | ✅ **boots and plays** — intro, campaign maps, combat |
| Op4 game libraries inside the XBE | ✅ 660 server + 71 client entry points, 0 unresolved |
| Save/restore | ⚠️ mechanism resolved, not yet exercised end to end |

### How the engine blocker was removed

Phase 2 was stuck on a real problem: nxdk has no PE loader, so the game `.dll`s could
not be loaded at runtime. The way out was to stop loading them at all — the game
libraries are **compiled with nxdk, archived, and linked into the `.xbe`**, with a
generated table mapping each export name to its symbol so the engine's
`COM_LoadLibrary`/`COM_GetProcAddress` resolve against memory instead of a filesystem
(§20, §24). A useful consequence: hlsdk's 660 `EXPORT`-marked think/touch/use handlers
resolve by name exactly as on Windows, so save/restore needs neither Source-style
datamaps nor the offset-based scheme validated in Phase 1.

### What the log is actually about

Most of `BUILD.md` is not build instructions. It is the record of hunting bugs on
hardware with no debugger, where the only witnesses are a log file written to the disc
and what appears on a CRT. A few that took the longest:

- **Every NPC facing the wrong way**, which turned out to be two `%g`/`%f` format
  specifiers feeding values into a parser — the same class of bug, in two places,
  breaking scripted sequences, doors and puzzles across the whole game (§53–54).
- **Decals invisible for thirteen sections**, chased through the renderer, the GL
  state, the palette upload and the NV2A's registers, and finally caused by a decal
  pool the fork had shrunk to 64 entries — smaller than the number of decals the maps
  place by themselves (§55–§68).
- **A GPU hang in combat**, decoded from a pbkit register dump down to a texture whose
  format word described a texture with zero dimensions (§72).
- Several traps that produce a *silently wrong binary* rather than an error: a
  packaging step that fed on its own previous output, and a "forced" relink that never
  actually relinked because waf hashes content and `touch` changes nothing (§62, §72).

The log is written in Spanish and is deliberately verbose: it records the dead ends as
carefully as the successes, and it marks in every section what has **not** been
verified.

---

## What's in here

```
BUILD.md                     the engineering log: every command, every failure, every fix
patches/engine/              local patches against xash3d-fwgs (GPLv3)
patches/hlsdk/               local patches against hlsdk-portable (see LICENSE-NOTE.md)
patches/pbgl/                local patches against pbgl            (see LICENSE-NOTE.md)
LICENSE                      GPLv3
```

---

## Reproducing

Everything is in **`BUILD.md`**. In short:

1. §1 — enable i386 multiarch and install the 32-bit toolchain (Ubuntu/WSL).
2. §2–§3 — build the engine and the Opposing Force game libraries, 32-bit.
3. §6 — copy `valve/` and `gearbox/` from **your own legal copy** of the games.
4. §7 — launch with `-game gearbox`.

For the Xbox side: §17 covers installing and verifying nxdk and pbgl, §18 and §24 cover
building the game libraries with nxdk and linking them into the `.xbe`, and §21–§22
cover getting the result onto a console and reading its log back over FTP.

The patches are not upstreamed. §11 explains what each one does and whether it is
mandatory or only needed for a specific experiment.

---

## Licensing

This repository is **multi-provenance on purpose**. Read this before reusing anything.

### The repository itself — GPLv3

`LICENSE` (GPLv3) covers this repository's own content: `README.md`, `BUILD.md`, and
the patches under **`patches/engine/`**, which are derived from
[Xash3D FWGS](https://github.com/FWGS/xash3d-fwgs) and therefore inherit its GPLv3.

### `patches/hlsdk/` — NOT GPLv3

Derived from [hlsdk-portable](https://github.com/FWGS/hlsdk-portable), a fork of the
**Half-Life SDK by Valve**, and governed by that SDK's own terms. See
[`patches/hlsdk/LICENSE-NOTE.md`](patches/hlsdk/LICENSE-NOTE.md).

### `patches/pbgl/` — MIT

Derived from [pbgl](https://github.com/fgsfdsfgs/pbgl), which is MIT. See
[`patches/pbgl/LICENSE-NOTE.md`](patches/pbgl/LICENSE-NOTE.md).

Putting the GPLv3 `LICENSE` at the repository root does **not** relicense the hlsdk or
pbgl patches, and no such relicensing is claimed or intended.

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
  this work builds on
- [@fgsfdsfgs](https://github.com/fgsfdsfgs) — [pbgl](https://github.com/fgsfdsfgs/pbgl),
  OpenGL 1.x for the Xbox over nxdk/pbkit
- [XboxDev](https://github.com/XboxDev) — nxdk
- [@brentdc-nz](https://github.com/brentdc-nz) — Half-LifeX, the reference for what
  running Half-Life on this hardware actually costs
