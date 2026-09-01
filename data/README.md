# Crash2Fix Dataset and Analysis

Data, validation results, and documentation for the **Crash2Fix Firefox crash-bug benchmark**.

## Shared Documents

The same materials are available on Google Drive:

- **Bug validation results (spreadsheet)**  
  https://docs.google.com/spreadsheets/d/1-OMABZUObrV_aHDE9odBT26IWRNakq_v/edit

- **Pipeline analysis and documentation (document)**  
  https://docs.google.com/document/d/1UO77cDq_Eq54PcwlgMHFlzLEmjAp1qHOWWxt68wGHBE/edit

## Dataset Summary

The benchmark contains **230 unique Firefox crash bugs** collected from Bugzilla and processed through our crash reproduction and validation pipeline.

### Validation Results

| Dataset Partition | Bugs | Validated | Validation Rate |
|----------|------:|----------:|---------------:|
| Prebuilt_Artifact | 27 | 13 | 46.2% |
| Full_Build_Successful | 56 | 21 | 37.5% |
| Failed_Builds | 150 | — | — |

A bug is considered **validated** when the discovered test **passes on the fixing commit and fails on its parent commit**.

## Repository Contents

### `bugzilla-bugs-report.xlsx`

Build and test-validation results for all analyzed bugs.

Sheets:

- **Prebuilt_Artifact**: Validation results using Mozilla-provided build artifacts.
- **Full_Build_Successful**: Validation results obtained from successful local builds.
- **Failed_Builds**: Build failures grouped by root cause.

The most common build failure is a **cbindgen TOML duplicate-key parsing error**.

### `Bugzilla Bug Analysis (Build & Test Validation).pdf`

Detailed documentation of the end-to-end benchmark construction and validation pipeline, including:

1. Crash bug extraction
2. Stack trace collection
3. Regression range filtering
4. Fixing-commit identification
5. Parent-commit identification
6. Test discovery
7. Build readiness checks
8. Artifact-based and full-build validation

The document also discusses unresolved issues encountered during validation, including:

- cbindgen build failures
- Framework-specific dependency discovery challenges

## Experimental Environment

All experiments were conducted on:

- **Platform:** Chameleon Cloud (KVM@TACC)
- **Operating System:** Ubuntu 22.04
