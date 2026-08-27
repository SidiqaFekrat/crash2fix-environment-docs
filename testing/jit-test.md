# jit-test - Requirements 

**Official docs:** https://firefox-source-docs.mozilla.org/js/test.html

jit-test runs SpiderMonkey's internal JavaScript engine test suite, verifying core JS language correctness and JIT compiler behavior
(Baseline, Ion, etc.) under different optimization configurations.

## Requirements

- **`dist/bin/js`** (the standalone SpiderMonkey shell) must exist in
  the build — requires `--enable-js-shell` in mozconfig at build
  time. Saved builds made before this flag was added won't have it:
```bash
  ls -la objdir-fullBuild/dist/bin/js
```
  If missing, the build needs to be redone with the flag in place —
  not fixable by retesting alone.

- No extra runtime packages beyond the general checklist — jit-test
  runs the JS shell directly, no browser/display involved.

## Path format is the real recurring problem

`./mach jit-test` matches test paths by substring, per Mozilla's own docs. In practice, we found that passing the full repo-relative path stored in our manifest (e.g. `js/src/jit test/tests/asm.js/bug1924062.js`) fails silently — no error, `returncode: 0`, empty stdout, "No tests found matching command line arguments." Passing the path relative to
`js/src/jit-test/tests/` instead (e.g. `asm.js/bug1924062.js`) works reliably. The practical fix: strip the prefix before invoking.

**Pipeline impact:** this affects every jit-test bug in the manifest, since `mach_command` is generated the same way for all of them. Our generic result parser defaults differently for `fixing` vs. `parent` role when stdout is empty (`fixing` → `"fail"`, `parent` → `"pass"`), producing exactly backwards results for a test that never actually ran on either side.

## Parent runs can fail on a missing shell flag, not the bug

Patching the test file onto the parent commit isn't always enough. A jit-test declares the shell flags it needs in a `|jit-test|` header line, and when a fix introduces both a new flag and a test that uses it, the parent commit's JS shell doesn't recognise the flag.

The shell prints its full option list, then:
```
Error: Invalid long option: --disable-decommit
Exit code: 1
FAILURES:
gc/bug-1940692.js
```
FAILURES:` is present, so a naive parser scores this `fail` — and since `fail` is the expected parent outcome, the pair looks validated. It isn't: the test never executed.

**Exit code is what separates the two cases.** A genuine reproduction at the parent gives `Exit code: -11` (SIGSEGV). A missing flag gives `Exit code: 1` alongside `Invalid long option:`. Both print `FAILURES:`, so the exit code and the error string are the only reliable signals.

**Parser handling** — check before any pass/fail logic, since `FAILURES:` would otherwise match first:

```python
if "Invalid long option:" in stdout:
    return "technical"
```

**Not fixable by patching more.** The flag comes from the fixing commit's C++ changes, not from any file that can be pulled across with `hg cat`. Record as `technical` and exclude the bug.

Observed on bug 1940692 (`--disable-decommit`, parent `e3f2eed0`).


## Useful Flags (from Mozilla's docs)

- `--jitflags=<bundle>` — test under specific JIT configurations (`all`, `interp`, `none`, etc.)
- `-R <filename>` / `--retest <filename>` — rerun only previously-failed tests
- `-s` / `--show-cmd` — print the exact shell invocation per test

## Reading the Output

| Output | Meaning |
|---|---|
| `PASSED ALL` | Every JIT configuration variant passed |
| `Exit code: -11` | SIGSEGV (segfault) — real crash under one or more JIT variants |
| `No tests found matching command line arguments.` (stderr, returncode 0) | Path format problem, not a real test outcome — see above |

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `No tests found matching command line arguments.`, empty stdout, `rc=0` | Full repo path passed instead of path relative to `js/src/jit-test/tests/` | Strip the prefix (see above) |
| Identical empty-output bug producing opposite pass/fail between fixing and parent | Side-effect of the path bug — neither side actually ran | Same fix; rerun with corrected path |
| `dist/bin/js: No such file or directory` | Saved build predates `--enable-js-shell` | Needs a fresh build with the flag in mozconfig |
| `Exit code: -11` reproduces identically on rerun | Genuine crash (e.g. wasm/asm.js memory handling) | Confirmed real result — investigate via the bug's actual ticket, not environment |
