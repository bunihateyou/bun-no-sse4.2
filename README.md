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

## Running programs without SSE4.2

The Bun binary alone isn't always enough — programs built *on top of* Bun can ship their own prebuilt SSE4.2 binaries that need separate attention. Below are instructions for two such programs.

### <sub>`omp`</sub> (Oh-My-Pi)

Repo: [`can1357/oh-my-pi`](https://github.com/can1357/oh-my-pi)

`omp` depends on `@oh-my-pi/pi-natives`, a Rust/N-API native addon shipped as a prebuilt `.node` binary. Even the `"baseline"` variant is compiled at `target-cpu=x86-64-v2` (SSE4.1/SSSE3) and SIGILLs on K10 when loaded. You must rebuild it from source at `target-cpu=barcelona`:

```sh
git clone --depth 1 https://github.com/can1357/oh-my-pi.git /tmp/oh-my-pi
cd /tmp/oh-my-pi

# RUSTFLAGS overrides omp's build script (which defaults to x86-64-v2/v3).
# The script only sets RUSTFLAGS when unset, so pre-setting wins.
RUSTFLAGS="-C target-cpu=barcelona" cargo build --release -p pi-natives

# Replace every prebuilt baseline .node in the global install:
find ~/.bun/install/global -name 'pi_natives.linux-x64-baseline.node' \
  -exec cp target/release/libpi_natives.so {} \;
```

> **After any `omp update`:** the SSE4.1 `.node` comes back from npm. Re-run the `cp` line above. The `modern` variant can be ignored — it's not loaded on non-AVX2 CPUs.

---

### <sub>`claude-code`</sub> (Claude Code)

Repo: [`anthropics/claude-code`](https://github.com/anthropics/claude-code)

Claude Code ships as a `bun build --compile` standalone binary — a 238 MB ELF with an embedded Bun runtime + JavaScriptCore baked in at modern ISA. It SIGILLs on K10 and can't be patched in-place. However, the bundled JS source can be **extracted** from the binary and run with our barcelona Bun instead:

```sh
# 1. Download the native binary (from npm — it's the same binary the curl installer gives)
mkdir -p ~/claude-code && cd ~/claude-code
curl -sL -o cc.tgz "https://registry.npmjs.org/@anthropic-ai/claude-code-linux-x64/-/claude-code-linux-x64-$(curl -s https://registry.npmjs.org/@anthropic-ai/claude-code/latest | grep -o '"version":"[^"]*"' | cut -d'"' -f4).tgz"
mkdir -p cc-extract && tar xzf cc.tgz -C cc-extract
# the native binary is at cc-extract/package/claude

# 2. Extract the bundled JS source using bun-extractor
curl -sL -o extract.py https://raw.githubusercontent.com/lauralex/bun-extractor/main/bun_extractor.py
python3 extract.py cc-extract/package/claude
# → outputs /tmp/claude_extracted/root/src/entrypoints/cli.js (the full Claude Code source, ~18 MB)

# 3. Install the extracted source + a wrapper that runs it with our bun
cp /tmp/claude_extracted/root/src/entrypoints/cli.js ~/claude-code/cli.js

cat > ~/claude-code/claude <<'EOF'
#!/bin/sh
export DISABLE_INSTALLATION_CHECKS=1
export DISABLE_UPDATES=1
exec "$HOME/bun-barcelona/bun" "$HOME/claude-code/cli.js" "$@"
EOF
chmod +x ~/claude-code/claude
ln -sf ~/claude-code/claude ~/.local/bin/claude

# 4. Two patches needed in cli.js (the endpoint/model compatibility fixes):
#    a) Skip model validation (endpoint 404s on model IDs with slashes)
perl -i -pe 's/retrieve\(e,t=\{\},n\)\{let\{betas:r\}=t\?\?\{\};return this\._client\.get\(Ea`\/v1\/models\/\$\{encodeURIComponent\(e\)\}\?beta=true`,\{\.\.\.n,headers:hs\(\[\{\.\.\.r\?\.toString\(\)!=null\?\{"anthropic-beta":r\?\.toString\(\)\}:void 0\},n\?\.headers\]\)\}\)\}/retrieve(e,t={},n){return Promise.resolve({id:e,object:"model",display_name:e,created:0,owned_by:"gateway"})}/g' ~/claude-code/cli.js
perl -i -pe 's/retrieve\(e,t=\{\},n\)\{let\{betas:r\}=t\?\?\{\};return this\._client\.get\(Ea`\/v1\/models\/\$\{encodeURIComponent\(e\)\}`,\{\.\.\.n,headers:hs\(\[\{\.\.\.r\?\.toString\(\)!=null\?\{"anthropic-beta":r\?\.toString\(\)\}:void 0\},n\?\.headers\]\)\}\)\}/retrieve(e,t={},n){return Promise.resolve({id:e,object:"model",display_name:e,created:0,owned_by:"gateway"})}/g' ~/claude-code/cli.js
#    b) Convert mid-conversation system messages to user-role (some endpoints reject role:"system" in messages)
perl -i -pe 's/if\(f\.type==="api_system"\)return\{role:"system",content:f\.message\.content\}/if(f.type==="api_system")return{role:"user",content:[{type:"text",text:"<system-reminder>"+(typeof f.message.content==="string"?f.message.content:JSON.stringify(f.message.content))+"<\/system-reminder>"}]}/g' ~/claude-code/cli.js

# 5. Verify
claude --version   # → 2.1.x (Claude Code)
```

> **Note:** Claude Code's bundled source uses a `bun build --compile` format with a `\n---- Bun! ----\n` trailer. The extractor reads this format directly. The two `perl` patches fix: (a) model-ID-with-slashes causing a 404 on `/v1/models/{id}` validation, and (b) `role:"system"` in the messages array being rejected by some Anthropic-compatible endpoints. Neither patch affects functionality — they only bypass client-side validation and rewrap system messages.
>
> **After updating Claude Code:** re-run steps 1-4 to extract + patch the new version. The `DISABLE_UPDATES=1` env var in the wrapper prevents the auto-updater from replacing your setup silently.

## Why not just use QEMU?

You can run stock Bun under `qemu-x86_64 -cpu max`, but TCG emulation is ~10-20x slower and uses A LOT OF CPU. This native build runs at full speed on K10 hardware.

## License

Bun is MIT-licensed. WebKit is BSD/LGPL. This repo contains only the build workflow + README — no Bun source. Download the binary from [Releases](../../releases).
