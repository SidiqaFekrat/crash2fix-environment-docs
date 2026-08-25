## Building in a Python 3 Environment

### 1. Clone mozilla-central
```bash
hg clone https://hg.mozilla.org/mozilla-central
```

### 2. Install Mercurial (if not already present)
```bash
sudo apt-get update
sudo apt-get install -y mercurial
```

### 3. Update to tip before bootstrapping
`./mach bootstrap` installs dependency versions matched to whatever commit is currently checked out on mozilla-central. Since different historical commits require different toolchain versions, always bootstrap from tip not from whatever commit you'll build later - to get a consistent baseline before pinning versions manually per commit.

```bash
hg update -r tip --clean
```

### 4. Run mach bootstrap
```bash
./mach bootstrap
```
This detects the available Python version, creates a virtualenv at '~/.mozbuild/_virtualenvs/mach/bin/python', and installs mach's own dependencies into it.

### 5. mozconfig
```
ac_add_options --disable-bootstrap
ac_add_options --without-wasm-sandboxed-libraries
mk_add_options MOZ_OBJDIR=./objdir-fullBuild
mk_add_options AUTOCLOBBER=1
mk_add_options MOZ_MAKE_FLAGS=-j4
ac_add_options --enable-js-shell
export CC=/usr/bin/clang-17
export CXX=/usr/bin/clang++-17
export CBINDGEN=/home/cc/.cargo/bin/cbindgen
```

| Option | Purpose |
|---|---|
| `--disable-bootstrap` | Prevents `mach` from auto-updating toolchains mid-build, so we retain full control over installed versions |
| `--without-wasm-sandboxed-libraries` | Skips WASM sandbox library builds — not needed for a research/testing environment |
| `MOZ_OBJDIR=./objdir-fullBuild` | Names the full-build output directory distinctly from artifact-build output |
| `AUTOCLOBBER=1` | Auto-cleans the build dir when configuration changes between commits, avoiding stale-artifact errors |
| `MOZ_MAKE_FLAGS=-j4` | Caps parallel compile jobs to 4, balancing speed against available memory |
| `--enable-js-shell` | Builds the standalone SpiderMonkey JS shell binary, required for running jit-test |
| `CC` / `CXX` | Explicitly pins the compiler to clang/clang++ 17, overriding whatever `mach bootstrap` installed for tip |
| `CBINDGEN` | Explicitly pins cbindgen to the cargo-installed 0.27.0 binary, overriding bootstrap's newer default |


### 6. Toolchain version pinning

`./mach bootstrap` installs whatever toolchain versions **tip** currently requires, which are often newer than what historical commits need. Bootstrap is also interactive by design - it prompts for choices (build target, e.g. "Firefox for Desktop"; a Mercurial configuration wizard; telemetry opt-in) that expect a human answering at a terminal. Since our pipeline builds hundreds of historical commits automatically and unattended, running full interactive bootstrap once per commit is both impractical and unnecessary.

Instead, bootstrap is run **once**, from tip, to establish the base toolchain and system dependencies. From there, only the specific toolchain versions known to vary meaningfully across commits - clang, cbindgen, and Rust - are pinned explicitly per build via `mozconfig`, rather than re-running bootstrap itself for every commit. This assumes a single, shared build environment across all commits, with per-commit variation handled entirely through the pinning below. It's possible other bootstrap-installed dependencies (system packages, Node, nasa, etc.) have also shifted somewhere across our 2018-2026 commit range in ways we haven't yet needed to account for - this is a known limitation of the current approach, not something we've fully ruled out.

**clang** - upgraded from the system default (14.0.0) to 17.0.6.
Newer Firefox commits require clang/LLVM 17.0+ and abort configure with 'Only clang/llvm 17.0 or newer is supported' otherwise.
```bash
wget https://apt.llvm.org/llvm.sh
chmod +x llvm.sh
sudo ./llvm.sh 17
```

**Rust** - rather than pinning one specific version, we set the toolchain to always resolve to the latest stable release, since different commits have different minimum requirements (>=1.82.0):
```bash
echo 'export RUSTUP_TOOLCHAIN=stable' >> ~/.bashrc
source ~/.bashrc
```
confirmed working version: rustc 1.97.0.

**cbindgen** - upgraded 0.26.0 to 0.27.0:
```bash
source $HOME/.cargo/env
cargo install cbindgen --version 0.27.0 --force
```

### 7. Known build failure: missing libstdc++ headers

**Symptoms:**
```
fatal error: `cstddef` file not found
The libstdc++ in use is not new enough
```

**Cause:** clang 17 depends on the system's GCC-provided libstdc++ headers for the C++ standard library, but these were not installed alongside clang 17 on a fresh Chameleon Cloud instance.

**Fix:**
```bash
sudo apt-get install libstdc++-12-dev
```
This installs the missing headers under '/usr/include/c++/12/'.
After installation, `configure`'s "checking for new enough STL headers from libstdc++" check passes.
