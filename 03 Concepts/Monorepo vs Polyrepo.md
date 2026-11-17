---
tags: [concept]
---

# Monorepo vs Polyrepo

Up: [[Concepts MOC]]

## The idea

A monorepo keeps many projects in one version-control repository. A polyrepo gives each project its own. Monorepos make cross-project changes atomic and share tooling; polyrepos give clear ownership, small clones and independent release cycles.

Neither is better in general. The choice depends on team size, how often projects change together, and how much tooling the organisation will maintain.

Key points:

- **Monorepo**: one commit can change a backend and its client together.
- **Polyrepo**: each repo has its own CI, branches and versioning.
- **Coupling**: polyrepos need versioned contracts between projects.
- **Tooling**: monorepos need build caching and path-based CI to stay fast.

## In this codebase

Polyrepo, with one deliberate shared-code exception. Each folder in the workspace is its own git clone with its own GitLab remote (checked with `git remote get-url origin`):

| Folder | Remote group |
|---|---|
| `tgmdp_backend`, `tgmdp_jobs` | `gitlab.orbit-poc.com/tgmdp/server/` |
| `tgmdp_web_frontend`, `tgmdp_vendor_frontend` | `.../tgmdp/web/` |
| `tgmdp-app-ams`, `tgmdp_vendor_rn` | `.../tgmdp/app/` |
| `tgmdp_rn_app`, `tgmdp_fake_esb`, `niws2-app-legacy`, `ams2-app-legacy` | personal-namespace repos |

- **Shared code by git submodule.** `tgmdp_backend/.gitmodules` and `tgmdp_jobs/.gitmodules` both point `sharedModules` at one repo, `tgmdp_shared_modules.git`; `settings.gradle` then `include 'sharedModules'` (backend line 6, jobs line 2). This is the middle path: separate deployables, one shared entity/repository library pinned by commit (backend currently at a `SIT_20260918-...` describe tag). Cost: a change to a shared entity needs a commit in the submodule repo, then a pointer bump in each consumer. A stale pointer is a classic source of "works in backend, breaks in jobs".
- **Contracts between repos are informal.** The web consoles and the RN shell call the backend through HTTP and the ESB through WSDLs; there is no generated client shared across repos, so the contract is by convention (see [[Contract Testing]], [[API Versioning]]: `/api/v1/<app>/`).
- **Release independence is the point.** The AMS web module is built by its own CI and published as a zip to the backend's OTA registry, then the shell downloads it; no app-store release needed. See [[Over-the-Air Updates]] and [[Hybrid Shell Architecture Trade-offs]].
- **The workspace itself.** The `NWIS_Legacy/` folder is a separate git repo that holds docs, reports and the parity reviews. The `legacy/` folder has no `.git` of its own at the top, only two nested ones (`legacy/TGMDP_legacy_new`, `legacy/tgmdp_vendor_legacy`).
- **Legacy.** The old TGMDP monolith was effectively one big source tree: JSP, actions, DAOs and the `ADS` adapters in sibling directories under `legacy/`. See [[Legacy Systems]].

Practical effect: a feature that spans backend and web console means two branches, two merge requests and an ordering decision (deploy backend first).

## Why it matters

A change that spans backend and frontend means coordinated branches, merges and releases across repos.

## Check yourself

You change a column in a shared entity. List the repos you must touch and in what order.

## Related

- [[Git Branching]] · [[API Versioning]] · [[Code Review]] · [[Layered Architecture]] · [[Strangler Fig]]
- [[tgmdp_backend]] · [[tgmdp_jobs]] · [[Concepts MOC]]
