# GTest — Requirements 

**Official docs:** https://firefox-source-docs.mozilla.org/gtest/index.html

## Targeted gtest runs fail after restoring a saved build

Per Mozilla's own docs, `xul-gtest` (the library containing gtest binaries) is only built "when tests are required," not as part of a normal build — and `mach gtest` is documented to rebuild it automatically before running. In practice, running `./mach gtest <filter>` directly against a freshly-restored saved build instead triggers an unreliable targeted/incremental build path and fails:

```
Hey! Builds initiated with mach build $A_SPECIFIC_TARGET may not always work,
even if the code being built is correct. Consider doing a bare mach build instead.
Could not build xul-gtest
```
**Fix:** run a bare `./mach build` first, then the gtest filter separately — don't rely on `mach gtest` to rebuild on its own after a saved-build restore.

**Detection signal for automation:** `"Could not build xul-gtest"` in stdout means the rebuild step was skipped — not a real test failure, not a genuine/technical result. Treat this the same way as CLOBBER staleness: a recurring per-restore requirement specific to this framework.

**Note:** the automated pipeline's `mach_build()` always runs a full build before any test, so this shouldn't occur there — it's specific to manually restoring a saved build and jumping straight to `./mach gtest` without rebuilding first.

## Requirements

- No display/Xvfb needed — gtest has no browser window.
- - `MOZ_RUN_GTEST=1` with `./mach run --debug` if debugging a specific failing gtest interactively (per Mozilla's docs — remember to `mach gtest` first to relink, since this isn't part of a top-level build).

## On a fresh instance, gtest's build step can trigger `configure` from scratch

If `xul-gtest` was never linked before, `mach gtest`'s build-then-run step means a completely fresh `configure` — surfacing a long chain of system dependencies a saved-build restore would otherwise never reveal. Confirmed on this session, in the order encountered:

1. **llvm-objdump**
```bash
   sudo apt-get install -y llvm-17
   sudo ln -sf /usr/bin/llvm-objdump-17 /usr/bin/llvm-objdump
```
2. **pkg-config**
```bash
   sudo apt-get install -y pkg-config
```
3. **alsa** — the `dev` package specifically; the runtime `libasound2` alone is not enough
```bash
   sudo apt-get install -y libasound2-dev
```
4. **libpulse**
```bash
   sudo apt-get install -y libpulse-dev
```
5. **clang/libclang for bindgen** — needs both binary and library, plus unversioned symlinks since LLVM's apt packages version-suffix everything
```bash
   sudo apt-get install -y libclang-17-dev
   sudo ln -sf /usr/bin/clang-17 /usr/bin/clang
   sudo ln -sf /usr/bin/clang++-17 /usr/bin/clang++
   sudo ln -sf /usr/lib/llvm-17/lib/libclang-17.so /usr/lib/llvm-17/lib/libclang.so
   sudo ldconfig
```
**GTK/graphics stack** (pango, cairo, harfbuzz, fontconfig, freetype resolved together)
```bash
   sudo apt-get install -y libgtk-3-dev libxkbcommon-dev
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Could not build xul-gtest` | Targeted build attempted on a freshly-restored saved build without a full rebuild first | Run full `./mach build` before the gtest filter command |
| `TEST_END: Test CRASH` but subtest count and summary show full pass | Possible shutdown/teardown crash, not a real test failure — unconfirmed | Rerun to check reproducibility before concluding |


