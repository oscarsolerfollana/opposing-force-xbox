# Licensing note for `patches/pbgl/`

**The patch in this directory is NOT covered by the GPLv3 `LICENSE` at the root of this
repository.**

It is a diff against [fgsfdsfgs/pbgl](https://github.com/fgsfdsfgs/pbgl), a partial
OpenGL 1.x implementation for the original Xbox on top of nxdk/pbkit, which is
distributed under the **MIT License** (Copyright (c) 2020 fgsfds). Anything derived from
it — including this patch — is governed by those terms.

- Upstream license: <https://github.com/fgsfdsfgs/pbgl/blob/master/LICENSE>

MIT and GPLv3 are compatible in one direction only, and placing a GPLv3 `LICENSE` file
at the repository root does not relicense these changes. If you reuse them, keep the MIT
notice.

## What this patch is

| File | Target | What it does |
|---|---|---|
| `pbgl-xbox-fixes.patch` | `src/array.c`, `src/memory.c`, `src/misc.c`, `src/pbgl.c`, `src/state.c`, `src/texture.c` | The accumulated fixes and instrumentation this port needed from pbgl. |

The changes fall into three groups, all described in `BUILD.md`:

- **Correctness fixes** for cases the upstream library did not hit. The most important
  is the texture-unit guard: pbgl programmed the NV2A's texture registers without
  checking that the bound texture had ever received data, and a texture object with zero
  dimensions makes the GPU raise an error and halt the console. The guard refuses to
  program such a unit and logs it instead (§72).
- **A hang fix** in the paletted-mipmap path found in §40–41.
- **Debug instrumentation**: a state dumper (`pbgl_dbg_state`) that prints the real GL
  state at a chosen draw call, and allocation accounting for the contiguous-memory pool
  the library carves textures out of. These were essential to the decal investigation of
  §55–§67 and are cheap enough to leave in.

Every hunk carries a comment marking it as **not upstream** and pointing at the
`BUILD.md` section that explains why it exists.

## Scope

This patch touches **library source only**. It contains no Half-Life engine or game
code, and no game assets.
