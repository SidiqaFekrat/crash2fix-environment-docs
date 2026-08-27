# Mochitest & Mochitest-Browser - Requirements 

Covers plain, chrome, and browser flavors.

**Official docs:** 
- Plain/chrome: https://firefox-source-docs.mozilla.org/testing/mochitest-plain/index.html
- Browser: https://firefox-source-docs.mozilla.org/testing/browser-chrome/index.html

Both covered command syntax, flavors, and test-writing accurately.
Neither covers what's needed to actually run mochitest against arbitrary historical commits - reconstructing the right toolchain, patching in test files and manifest entries that didn't exist yet at older revisions, and other environment gaps we only found running into them. That's what's documented below.

## Requirements 

No mochitest-specific runtime packages beyond the general checklist:
- **libgtk-3-0** - Firefox's core XPCOM loading depends on GTK3 even in headless mode; without it, the binary fails to start entirely (`libgtk-3.so.0: cannot open shared object file`, a well-documented issue across multiple Mozilla Bugzilla reports).
-  **xvfb + DISPLAY** - standard requirement for running any Firefox automation on a linux server without a physical display.
-  **gsettings-desktop-schemas** - resolves a "No such schema" startup error we hist directly on a fresh instance.

## From Mozilla's Own Docs (worth restating here)

Browser-flavor tests use real focus and input. Mozilla's docs explicitly warns: "you'll need to not touch your machine while running them," and that `--headless` avoids this but "some tests may break in this mode." Since our pipeline runs fully headless and unattended by necessity, we haven't independently verified which specific tests are affected - we're flagging this as as known, documented risk that may explain some of our unexplained test failures, not something we've isolated ourselves.

## Known Issues We Found

### Companion scripts are not runnable tests

File like `chromScript.js`, loaded via `SpecialPowers.loadChromScript()`, are helper code - not tests.
Running mach directly on one fails with "could not find any mochitest." Skip these; run only the actual test file.

### Parent commit: test files patched in, but still not found

mochitest discovers tests via a hand-maintained manifest (`mochitest.toml`/`.ini`), committed alongside the test - unlock WPT, this manifest is not auto-regenerated. If a fixing commit adds a test and its manifest entry together, the parent commit has neither. The test file must be patched in with `hg cat`, **and** the manifest entry must be patched in separately, or Mac still reports no tests found.

```bash
hg cat -r <fixing_commit> <test_file> > <test_file>
hg cat -r <fixing_commit> <manifest_path> > <manifest_path>
```

## Flavors

- **plain** — content scope, SpecialPowers access
- **chrome** — chrome scope, privileged APIs
- **browser** — browser window scope, real UI, needs real focus/input
- **a11y** — accessibility interfaces, no e10s/fission

**Useful flags:** `--enable-a11y-checks` · `--jsdebugger` ·
`--verify` (flush out intermittents) · `--setpref="pref=value"` ·
`-f browser --subsuite <name>` (run a whole subsuite, e.g.
`devtools`) · `--no-autorun` (pause before test starts, used with
`--jsdebugger`)

## Troubleshooting

| Error | Fix |
|---|---|
| `could not find any mochitests under ... <file>.js` | Pointed at a companion script — run the real test file instead |
| `hg revert`: `file not managed` | File didn't exist at this commit — `rm <path>` instead |
| Same failure reproduces identically on rerun | Genuine code/test issue, confirmed — not environment noise |

