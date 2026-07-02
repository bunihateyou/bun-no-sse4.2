**HELPFUL note**: IF you are using this on bun related projects, there might be components in them that have been shipped as binaries (with sse4.2 req) and those components would too need to be rebuilt by you. while this might be obvious this note might help some people.
# bun-no-sse4.2

A custom **Bun v1.4.0** build for x86_64 Linux CPUs **without SSE4.2 / SSE4.1 / SSSE3 / AVX** — e.g. AMD Athlon II X4, Phenom, Opteron K10, and similar pre-2008 microarchitectures.

Stock Bun ships a prebuilt WebKit/JavaScriptCore compiled at `-march=nehalem`, which emits `pcmpgtq` (SSE4.1) and `pshufb` (SSSE3) on JSC interpreter hot paths. On a K10-class CPU these trigger `SIGILL` (exit 132) the moment any JavaScript is evaluated — `bun -e "console.log(1)"` crashes. This build fixes that.

## What this build does differently

1. **`-march=barcelona`** (K10 baseline: SSE3, no SSE4.x/SSSE3/AVX) patched into [`scripts/build/flags.ts`](scripts/build/flags.ts) (C/C++ codegen) and [`scripts/build/rust.ts`](scripts/build/rust.ts) (Rust crates, `-Ctarget-cpu=barcelona`).
2. **WebKit built locally** (`--webkit=local`) instead of linking the prebuilt `-march=nehalem` `libJavaScriptCore.a` blob — JSC's hot paths are now SSE3-only.
3. **ICU 74 libs bundled** + an `LD_LIBRARY_PATH` wrapper, so the binary runs on distros whose system ICU differs from Ubuntu 24.04's (e.g. Void with ICU 78). `patchelf` is deliberately avoided — it segfaults Bun's non-standard ELF layout at runtime.

## Download & Install

```sh
# download the release tarball
wget https://github.com/bunihateyou/bun-no-sse4.2/releases/download/v1.4.0-barcelona/bun-barcelona-linux-x64.tar.gz

# extract to ~/.bun-barcelona
mkdir -p ~/.bun-barcelona
tar xzf bun-barcelona-linux-x64.tar.gz -C ~/.bun-barcelona

# always invoke the wrapper (sets LD_LIBRARY_PATH for the bundled ICU libs)
~/.bun-barcelona/bun --version   # → 1.4.0
~/.bun-barcelona/bun -e "console.log(1+2)"   # → 3
```

### Put `bun` on your PATH

The tarball ships a `bun` wrapper script that resolves its own location (follows symlinks) and sets `LD_LIBRARY_PATH` to the bundled `lib/` before exec'ing the real binary. Symlink it into a PATH dir:

```sh
ln -sf ~/.bun-barcelona/bun ~/.local/bin/bun   # or /usr/local/bin/bun
bun --version
```

> **Always call `bun` (the wrapper), never `bun-barcelona` directly** — the wrapper is what makes the bundled ICU 74 libs load. Calling the raw binary will fail with `libicui18n.so.74: cannot open shared object file` on distros without ICU 74.

### Verify it works on your CPU

```sh
bun -e "console.log(42)"
# 42  → good (exit 0)
# (crash / exit 132) → your CPU stock bun would SIGILL too; this build should fix it
bun -e "const a=new Int32Array([5,3,8,1,9]).sort(); console.log([...a])"
# [ 1, 3, 5, 8, 9 ]  (exercises the JSC typed-array path that previously SIGILL'd)
```

## Contents of the release tarball

```
bun             ← wrapper script (always invoke this)
bun-barcelona   ← the actual Bun v1.4.0 binary (58 MB, stripped, -march=barcelona)
lib/
  libicudata.so.74   (30 MB)
  libicui18n.so.74   (3.3 MB)
  libicuuc.so.74     (2.1 MB)
```

## Building from source (reproducing the release)

The GitHub Actions workflow at [`.github/workflows/build.yml`](.github/workflows/build.yml) builds this end-to-end on `ubuntu-latest` (~30-40 min). It:

1. Installs clang-21 + ninja + Rust nightly (pinned by Bun) + a bootstrap Bun
2. Clones `oven-sh/bun` and applies the two `-march=barcelona` patches via `sed`
3. Clones `oven-sh/WebKit` (blobless, pinned commit `c9ad5813…`) — the local WebKit build inherits `-march=barcelona`
4. Configures with `--webkit=local --baseline=true` and builds with ninja
5. Strips the binary, bundles ICU 74 libs + wrapper, uploads as an artifact

To trigger a build: push to `main`, or go to **Actions → "Build Bun (barcelona, no SSE4.2)" → Run workflow**.

### Building locally (not on GitHub Actions)

You need a machine **with** SSE4.2 (the build runs code generators that assume it), targeting a **barcelona** output:

```sh
git clone https://github.com/bunihateyou/bun-no-sse4.2.git && cd bun-no-sse4.2
# review the two patches already in .github/workflows/build.yml (the sed lines)
# then follow the same steps the workflow runs, substituting your paths
```

The two source patches (apply to a fresh `oven-sh/bun` clone):

```diff
# scripts/build/flags.ts  — C/C++ codegen target
-"-march=nehalem"
+"-march=barcelona"

# scripts/build/rust.ts  — Rust crate target (baseline ternary)
-? "nehalem"
+? "barcelona"
```

## Running `omp` (oh-my-pi) on a no-SSE4.2 CPU

Getting Bun running is necessary but **not sufficient** for `omp`. The `@oh-my-pi/pi-coding-agent` package depends on `@oh-my-pi/pi-natives`, a Rust/N-API native addon shipped as a prebuilt `.node` binary. Even the `"baseline"` variant (`pi_natives.linux-x64-baseline.node`, ~130 MB) is compiled at `target-cpu=x86-64-v2` (SSE4.1/SSSE3) and contains `pcmpgtq` + `pshufb` on default code paths → `SIGILL` on K10 when `process.dlopen` loads it.

You must rebuild `pi-natives` from source at `target-cpu=barcelona` and replace the prebuilt `.node`:

```sh
# on the target machine (or cross-build on a SSE4.2 host)
git clone --depth 1 https://github.com/can1357/oh-my-pi.git /tmp/oh-my-pi
cd /tmp/oh-my-pi

# RUSTFLAGS here OVERRIDES omp's build script (which would set x86-64-v2/v3).
# The script only sets RUSTFLAGS when it's unset, so pre-setting it wins.
RUSTFLAGS="-C target-cpu=barcelona" cargo build --release -p pi-natives

# the output is a cdylib: libpi_natives.so
# rename + replace every baseline .node in the global install
cp target/release/libpi_natives.so \
   ~/.bun/install/global/node_modules/@oh-my-pi/pi-natives-linux-x64/pi_natives.linux-x64-baseline.node
# there are nested copies under each @oh-my-pi/* package — replace them all:
find ~/.bun/install/global -name 'pi_natives.linux-x64-baseline.node' \
  -exec cp target/release/libpi_natives.so {} \;
```

Notes:
- The rebuilt `.node` still contains dispatched AVX2/SSSE3 SIMD (in image/sixel codecs) — these are behind runtime CPU-feature checks and never execute on K10. The non-dispatched `pcmpgtq`/`pshufb` that caused the SIGILL are gone.
- **Update fragility:** running `omp update` or `bun install -g @oh-my-pi/pi-coding-agent` again will re-fetch the SSE4.1 `.node` from npm and clobber the barcelona build. Re-run the `cp` above after any update until omp ships a v1 (SSE2) or barcelona variant.
- The `modern` variant (`.linux-x64-modern.node`) is not loaded on non-AVX2 CPUs (the loader's `detectHostAvx2Support()` picks `baseline`), so leaving it stock is fine.

## Why not just use QEMU?

You can run stock Bun under `qemu-x86_64 -cpu max`, but TCG emulation is ~10-20x slower and uses A LOT OF CPU. This native build runs at full speed on K10 hardware.

## License

Bun is MIT-licensed. WebKit is BSD/LGPL. This repo contains only the build workflow + README — no Bun source. Download the binary from [Releases](../../releases).
