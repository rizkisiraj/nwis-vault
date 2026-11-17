---
tags: [practice]
---
# Code Review

Up: [[Best Practices MOC]]

## The idea
Someone other than the author reads a change before it merges, to catch bugs, share knowledge and keep conventions. It works best with small changes, automated checks handling style, and humans spending attention on design and risk.

## Why it matters
- Second pair of eyes finds what the author cannot see.
- Review is where team conventions get taught.

## In this codebase
- **Merge-request flow:** history is full of `Merge branch 'fix/1064-photo-parity' into 'develop'` (`git log --merges` in `tgmdp_backend`), i.e. every change arrives through a GitLab MR from a short `fix/` or `feat/` branch. The RN repo adds a CI job `lint:commits` that runs only on `merge_request_event` (`tgmdp_rn_app/.gitlab-ci.yml:89-97`) to check commit author addresses.
- **Automation takes the cheap review work:** CI test jobs and the lint/credential/typecheck jobs in `tgmdp_rn_app/.gitlab-ci.yml:44-113` mean reviewers need not argue about style. `tgmdp_backend/.gitlab-ci.yml:246-247` and `:303-315` gate some jobs on MR events, one only when the target is `main`.
- **A written review artifact, parity reports:** `acpt_parity_review/` and `vendor_parity_review/` hold a Traditional-Chinese 新舊版比對檢查報告 per module, produced by the `parity-review` skill (`.claude/skills/parity-review/SKILL.md`), listing each discrepancy with legacy `file:line`, new `file:line`, severity and fix. This is review against legacy ground truth rather than against taste. Rules such as "a bare `.replace()` on a nullable projection getter is a finding on sight" (SKILL.md ~line 393-397) encode past incidents.
- **Decisions the reviewer cannot make alone:** `acpt_parity_review/TGMDPs009_工程驗收查詢作業_accepted-deviations.md` is a table (Type: Legacy bug / Design decision / Deliberate hardening) sent for business confirmation, so intentional differences are recorded rather than silently merged.
- **AI review step:** the `ai-review` and `parity-recheck` skills exist as a first pass and refresh a report against the latest `develop`/`development`.
- Gap: `tgmdp_vendor_frontend` and `tgmdp_web_frontend` have no CI lint/test gates of their own (web only includes an AutoQA job), so review is the main safety net there.

## Check yourself
Why record "accepted deviations" in a file instead of just approving the MR?

## Related
- [[Git Branching]] · [[Linting and Formatting]] · [[Clean Code Naming]] · [[Strangler Fig Migration and Legacy Parity]] · [[Best Practices MOC]]
