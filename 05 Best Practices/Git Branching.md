---
tags: [practice]
---
# Git Branching

Up: [[Best Practices MOC]]

## The idea
A branching strategy says where work happens and how it reaches production. Common shapes: trunk-based (short branches off `main`), or Git Flow (`main` plus an integration branch `develop`, with `feat/`, `fix/`, `hotfix/` branches). Pair it with a commit convention so history reads as a changelog.

## Why it matters
- Isolates unfinished work.
- Makes MR targets and CI rules predictable.

## In this codebase
Each repo picked its own integration branch.
- **Backend, documented Git Flow:** `tgmdp_backend/CONTRIBUTE.md:38-50` lists `main`, `develop`, `feat/*`, `fix/*`, `hotfix/*`; commits use `<type>(<scope>): <description>` (`:52-60`). Real branches follow it: `feat/H2-inbound-acpt-service`, `fix/TGMDPVENa002-audit-log`, and recent commits such as `fix(acpt): align TGMDPs012 with legacy behaviour`.
- **`develop` also in:** `tgmdp_rn_app`, `tgmdp_vendor_rn`, `tgmdp-app-ams` and `tgmdp_fake_esb` (default branch `develop`).
- **`development` instead:** `tgmdp_web_frontend` and `tgmdp_vendor_frontend`. Easy to mistype; the `parity-recheck` skill hard-codes this split (develop vs development per repo).
- **Branch name = module code or ticket number:** `fix/TGMDPs012-legacy-parity`, `fix/TGMDPVENb001-acceptance-parity`, `fix/1066-gnb-axios-timeout`, `fix/1076-irontower-layout-parity`. The number or code links the branch to a GitLab issue or a legacy module.
- **`tgmdp_jobs` is the loose one:** default `main`, with remote branches like `ChKPatorlToSuspend`, `Get_d2_data`, `FetchLogNoc` (no prefix), merged via `Merge branch 'SiteViewSync' into 'main'`. Evidence the convention is per-repo, not company-wide.
- Owner preference recorded in `tgmdp_backend/CLAUDE.md`: no `Co-Authored-By` trailers in commits.
- Legacy repos here are snapshots without a visible branch workflow.

## Check yourself
Which integration branch name does the vendor portal use, and why might scripts that assume `develop` break there?

## Related
- [[Code Review]] · [[Environment Separation]] · [[tgmdp_vendor_frontend]] · [[tgmdp_web_frontend]] · [[RN Shell Build and Environments]] · [[Best Practices MOC]]
