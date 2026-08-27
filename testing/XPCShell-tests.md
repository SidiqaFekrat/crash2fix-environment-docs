# xpcshell — Requirements 

xpcshell tests are unit-level tests with no browser chrome and low overhead, run via `./mach xpcshell-test`. Anything reachable from the XPCOM layer through scriptable interfaces is testable this way.


## Files that must be SKIPPED — not runnable standalone

Test filenames must start with `test_`; anything else in these directories is support code, not a test.

| Pattern | What it is | Result if run directly |
|---|---|---|
| `*_child.js` | Companion script loaded by `run_test_in_child()` from the real test | `UNKNOWN TEST` — never registered in the manifest on its own |
| `head.js` / `head_*.js` | Shared setup, loaded automatically via the manifest's `head` property | Same category as mochitest's `head_*.js` |
| `xpcshell.toml` / `.ini` | The manifest itself | `non-runnable extension` — already filtered by extension |

**`*_wrap.js` is NOT in this category.** Mozilla's docs describe the `test_foo.js` + `test_foo_wrap.js` pair explicitly: the wrapper calls `run_test_in_child()` to rerun the whole script in the child process, so the same logic gets exercised in both chrome and content. The docs' own child-debugging example invokes `test_simple_wrap.js` directly. These are real entry points — filtering them out loses coverage.

Note the asymmetry: filter `_child.js`, keep `_wrap.js`. Reference examples of the pattern live under `netwerk/test/unit_ipc/`.

## Manifest registration matters as much as file content

A test file existing on disk does not make it runnable. The `xpcshell.toml` manifest controls which tests are in the suite, and mach reports `UNKNOWN TEST` for anything not listed there.

To confirm a file is a registered, runnable test at a given commit:

```bash
grep -A3 "<filename>" $(dirname <filepath>)/xpcshell.toml
```

If it appears under a `[test_*.js]` header, it's registered.

**This bites specifically on parent-commit runs.** When a test was added alongside its fix, patching the `.js` via `hg cat` isn't enough — the parent's manifest may not list it yet:

```bash
hg cat -r <parent_commit> $(dirname <filepath>)/xpcshell.toml | grep -A3 "<filename>"
```

If the entry is missing, either the manifest needs patching the same way the test file does, or the parent-side run is untestable through this path and should be recorded as such.

## Requirements

- **Node.js must be installed.** Not documented as an xpcshell
  dependency anywhere upstream, but required in practice on a fresh
  instance:

```bash
sudo apt-get update
sudo apt-get install -y nodejs
```

  Point mach at it explicitly rather than running `mach bootstrap`, whose pinned `~/.mozbuild/node/` path has broader side effects:

```bash
MOZ_NODE_PATH=$(which node)
```

## Child-process tests may crash via IPC, unrelated to the fix

A `CHILD-TEST-STARTED` → minidump → `Exiting due to channel error` sequence with no assertions ever executed is an environment-level xpcshell IPC problem, not a verdict on the bug. Confirm scope by running an unrelated `_wrap.js` test:

```bash
find /data/FaultLocalizationIndustry/mozilla-central -path "*/firefox/*" -prune \
  -o -name "*_wrap.js" -path "*/test/unit*" -print | head -3
```

If an unrelated test crashes identically, it's environment-wide and out of scope to fix.

**Useful flags** (all from Mozilla's docs unless noted):
`--tag <tag>` run a tagged subset · `--sequential` force sequential execution, quieter output when reading logs (behavior visible in the docs' example output, flag not itself documented on that page) ·
`--jsdebugger` pause before running to attach the JS debugger on port 6000 · `--debugger <name> --debugger-interactive` for a C++ debugger ·
`--verify` to flush out intermittents (documented separately under Test Verification). `MOZ_DEBUG_CHILD_PROCESS=1` prints the child PID and sleeps so a debugger can attach — the one lever for the IPC issue above.

Default per-test timeout is 30 seconds, adjustable in-manifest via
`requesttimeoutfactor`.

## Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| `node not found at MOZ_NODE_PATH` | Node.js absent (fresh instance) | `sudo apt-get install -y nodejs`, then `MOZ_NODE_PATH=$(which node)` |
| `TEST_END: SKIP` treated as a pass | A `skip-if` condition in the manifest matched this environment | A skip is not a pass. Check `xpcshell.toml` for `skip-if` |
| `CHILD-TEST-STARTED` → minidump → `Exiting due to channel error`, no assertions ran | Environment-level xpcshell child-process IPC issue | Cross-check an unrelated `_wrap.js` test to confirm scope |
| `non-runnable extension (.toml)` | Manifest file, not a test | Already correctly skipped, no action |
| 0 tests run, exit 0 | Possible `--disable-tests` build, or a path that matched nothing | Check mozconfig first; this shape is dangerous because it isn't a failure |






