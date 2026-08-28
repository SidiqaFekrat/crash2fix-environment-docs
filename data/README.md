# Dataset and Analysis

Data and documentation for the Crash2Fix Firefox crash-bug benchmark.

## Files

### `bugzilla-bugs-report.xlsx` Per-bug build and test-validation results across 230 unique bugs. "Validated" means the test passed on the fixing commit and failed on its parent commit.

| Sheet | Bugs | Validated | % |
|---|---|---|---|
| `Prebuilt_Artifact` | 27 | 13 | 46.2% |
| `Full_Build_Successful` | 56 | 21 | 28.1% |
| `Failed_Builds` | 150 | — | — |

The `Failed_Builds` sheet clusters failures by root cause; the largest
cluster is the cbindgen TOML duplicate-key parse error.

### `Bugzilla Bug Analysis (Build &test validation).pdf`
Full pipeline documentation, Stage 1 through Stage 8: crash bug
extraction, stack trace collection, regression filtering, fixing-commit
and parent-commit identification, test discovery, build readiness,
artifact builds, and full builds split by Python era. Also contains the
open questions for Mozilla — the cbindgen build blocker and per-framework
dependency discovery.


Pipeline run on Chameleon Cloud (KVM@TACC, Ubuntu 22.04).
