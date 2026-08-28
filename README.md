# Firefox Build & Test Environment Notes

## About This Project

CrashToFix is a research project building a reproducible benchmark of validated Firefox crash-bug fixes, in the spirit of Defects4J and SWE-bench. For each bug, we pair its fixing commit against its parent (pre-fix) commit and validate the fix empirically: the bug's own test must fail on the parent commit and pass on the fixing commit.

Because the benchmark spans Firefox commits from 2018-2026, building and testing each commit requires reconstructing the correct toolchain and test-runtime dependencies for that specific point in Firefox's history - which shift considerably across this range. 


## About This Documentation

This repository captures the dependencies, toolchain pins, and setup steps we discovered while doing this - much of it through manual, pre-failure debugging on a fresh environment, since Mozilla's own 'mach bootstrap' covers build-time dependencies but not test-runtime ones. We're sharing it in case it's useful reference material, and we'd welcome corrections or pointers to existing Mozilla resources that cover this same ground.

## Repository Contents

| Path | Contents |
|---|---|
| `ENVIRONMENT_SETUP.md` | Base system setup, toolchain pins, and shared dependencies |
| `testing/` | Per-framework setup and run instructions, one file per test suite |
| `data/` | Bug-level build and test-validation results, and full pipeline documentation |






Compiled by Sidiqa Fekrat, CS Undergraduate, SABR Lab, DePaul University.
Advisor: Dong Jae Kim.
