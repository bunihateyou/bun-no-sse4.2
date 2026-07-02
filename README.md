# bun-barcelona-build

Builds a Bun runtime binary targeting `-march=barcelona` (AMD K10, SSE3 only — no SSE4.1/SSE4.2/SSSE3/AVX) so it runs on pre-Nehalem AMD CPUs (Athlon II, Phenom II, Opteron Gen 2-3) that Bun's official baseline (Nehalem, SSE4.2) cannot run on.

## Why

Bun requires SSE4.2 (Nehalem baseline, 2008+). AMD K10 CPUs (2007-2011: Athlon II, Phenom II, Opteron 23xx) have SSE3 + SSE4a + POPCNT but lack SSE4.1/SSE4.2/SSSE3/AVX — Bun crashes with SIGILL (exit 132).

## How

Two patches to oven-sh/bun:
1. `scripts/build/flags.ts`: `-march=nehalem` → `-march=barcelona` (C/C++/Zig codegen)
2. `scripts/build/rust.ts`: `target-cpu=nehalem` → `target-cpu=barcelona` (Rust crates)

Plus `--webkit=local` to rebuild WebKit/JavaScriptCore locally (the prebuilt baseline WebKit contains nehalem-emitted SSE4.1/SSSE3 instructions in JSC hot paths).

## Build

Trigger the GitHub Actions workflow (manual dispatch or push to main). The artifact `bun-barcelona` is uploaded — download and place on your K10 box.

```sh
# on the Athlon / Phenom II / Opteron:
chmod +x bun-barcelona
./bun-barcelona --version
```
