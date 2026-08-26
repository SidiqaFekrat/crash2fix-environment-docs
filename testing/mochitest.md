# Mochitest - Requirements & How to Run

Covers plain and chrome flavors (browser flavor intentionally excluded - see separate mochitest-browser doc, which shares the CLOBER and manifest-patching fixes below but has its own additional environment considerations around real browser window focus/input).

Official reference: https://firefox-source-docs.mozilla.org/testing/mochitest-plain/index.html
covers command syntax, flavors, debugging, and how to write tests accurately. What it doesn't cover is what's needed to actually run mochitest against arbitrary historical commits - reconstructing the right toolchain, patching in test files and manifest entires that didn't exist yet at older revisions, and other environment gaps we only found by running into them. That's what's documented below.

## Requirements

- **CLOBBER staleness after restoring a saved build** - checkout has moved through many commits since the build was saved:
```bash
touch objdir-fullBuild/CLOBBER
```
