# web-platform-tests (WPT) — Requirements 

**Official docs:** https://firefox-source-docs.mozilla.org/web-platform/index.html

WPT is a cross-browser testsuite ensuring browsers implement the same
behavior for web-platform features. Unlike mochitest, WPT's manifest
is automatically regenerated on every `mach wpt` run — no manual
manifest patching needed at parent commits.

## Requirements

- **`libgtk-3-0`** — confirmed required, Firefox won't launch without it.
- **`fontconfig`** — precautionary install; seen as a warning in our logs, not yet confirmed blocking.
```bash
  sudo apt-get install -y libgtk-3-0 fontconfig
```

## `typing_extensions` resets on every commit switch

A vendored `typing_extensions.py` inside WPT's third-party tooling
gets reset every time `hg update --clean` runs, breaking WPT's
Python tooling until reapplied:

```bash
cp $(find ~/.mozbuild/srcdirs/*/_virtualenvs/wpt -name "typing_extensions.py" -path "*/site-packages/*") \
   testing/web-platform/tests/tools/third_party/typing_extensions/src/typing_extensions.py
```
Must be reapplied after **every** commit switch, not just once. For
full pipeline automation this belongs inside the build script itself
(right after `hg_update()`, before `mach_build()`), not as a manual
step.

## WPT crashtests need a different command than in-tree crashtests

Files under `testing/web-platform/tests/.../crashtests/` are a
distinct WPT test type (alongside `testharness` and `reftest`) —
confirmed as a first-class WPT type, not a workaround. They **cannot**
run through `./mach crashtest` (the in-tree Gecko crashtest harness);
that command requires a manifest WPT crashtests don't use the same
way. The correct invocation is `./mach wpt` with `--test-type=crashtest`.

A WPT crashtest passing means only one thing: the page loaded and
rendered without crashing the browser — no pixel comparison, no JS
assertions. `TEST_END: PASS` = page survived; `TEST_END: CRASH` =
page crashed the browser.

For WPT crashtests specifically, replace command with:
```bash
./mach wpt --headless --no-pause-after-test --test-type=crashtest <test_filepath> 2>&1 | tee <bugid>_<role>_test.log
```
## Useful Flags (from Mozilla's docs)

- `--test-type=<testharness|reftest|wdspec|crashtest>` — restrict/select test type
- `--log-wptreport <file>.json` — structured output, also used by `mach wpt-update` to auto-generate expectation data
- `--meta testing/web-platform/meta` — use Firefox's expectation metadata
- `--no-pause-after-test` — required for automation; without it, a single testharness test leaves the browser open

## Troubleshooting

| Error | Fix |
|---|---|
| `ModuleNotFoundError: aioquic` | Just run `./mach wpt` once — self-bootstraps |
| `ImportError: cannot import name 'deprecated'` | Patch step above — reapply after every `hg update --clean` |
| `libgtk-3.so.0: cannot open shared object file` | `sudo apt-get install -y libgtk-3-0` |
| `fontconfig not available` | `sudo apt-get install -y fontconfig` if rendering issues appear |
| Any other `cannot open shared object file: <lib>` | `apt-cache search <lib>` → install matching package → retry |
| WPT crashtest run through `./mach crashtest` fails | Wrong harness — use `./mach wpt --test-type=crashtest` instead |

