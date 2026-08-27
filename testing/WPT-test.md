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

wpt starts a WebTransport-over-HTTP/3 server (`aioquic`), which pulls in pyOpenSSL, which imports `from typing_extensions import deprecated` — a decorator added in typing_extensions 4.5. The copy vendored at `testing/web-platform/tests/tools/third_party/typing_extensions/src/typing_extensions.py` predates it at commits in this range, so the import fails during server startup, before any test executes.

### Symptom

Traceback ending in `from typing_extensions import deprecated`, inside
`ensure_started` → `test_servers()`. `returncode 1`, roughly 100s spent
failing to start servers, no test result produced. Classify `technical` —
this is not a verdict on the bug.

### The fix

Copy the wpt venv's (newer) `typing_extensions.py` over the vendored one:

```bash
cp $(find ~/.mozbuild/srcdirs/*/_virtualenvs/wpt -name "typing_extensions.py" -path "*/site-packages/*") \
   testing/web-platform/tests/tools/third_party/typing_extensions/src/typing_extensions.py
```
## Why the copy doesn't stay applied — two observed failure modes
**1. The patch is reverted on every commit switch.** The vendored file is tracked by Mercurial, so `hg update --clean` restores Mozilla's version and discards the patch. Applying it once during environment setup is not enough. Verify with:

```bash
hg status testing/web-platform/tests/tools/third_party/typing_extensions/src/typing_extensions.py
# expect "M" — clean output means the patch is not in place
```

**2. The source of the copy may not exist yet.** `_virtualenvs/wpt/` is created lazily by mach on the first `./mach wpt` invocation. `./mach build` does not create it. On a fresh instance, or on the first wpt bug after one, `find` returns nothing and the copy silently does nothing — the test then runs against the unpatched vendored file and produces the traceback above.

Observed on bug 1885702: the freshly-built fixing commit failed on this import, while the restored parent commit — run afterwards, same source directory, same venv — passed. The venv had been created in the interim.

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

