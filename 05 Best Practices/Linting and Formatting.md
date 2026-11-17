---
tags: [practice]
---
# Linting and Formatting

Up: [[Best Practices MOC]]

## The idea
A **linter** finds bugs and risky patterns (unused vars, `==`, hardcoded passwords); a **formatter** fixes layout (spacing, import order) automatically. Run both in the editor, in a pre-commit hook and in CI, so nobody argues about style in review.

## Why it matters
- Style fights leave code review.
- Local hooks can be skipped (`--no-verify`), so CI must repeat the gates.

## In this codebase
- **Web console, ESLint only:** `tgmdp_web_frontend/eslint.config.js` (rules from line 106: `eqeqeq`, `prefer-const`, `no-var`, `camelcase`). No `.prettierrc` is in that repo; style is encoded as lint rules.
- **Vendor portal, ESLint + Prettier:** `tgmdp_vendor_frontend/.prettierrc` sets `tabWidth: 2`, `singleAttributePerLine`, and import sorting via `@trivago/prettier-plugin-sort-imports` plus `prettier-plugin-tailwindcss`. There is a `.prettierignore` too. The repo has no `.gitlab-ci.yml`, so nothing enforces this remotely.
- **RN shell, strictest:** `tgmdp_rn_app/package.json:34` runs `expo lint . --max-warnings 0`; `:36-39` add `lint:credentials`, `lint:deps`, `lint:android-config`, `lint:ios-config`; `:49` chains them as the `husky` script, run by `.husky/pre-commit`. `.gitlab-ci.yml` has matching jobs (`lint`, `lint:credentials`, `lint:deps`, `lint:commits`) and `.swiftlint.yml` covers Swift. Custom `no-restricted-syntax` selectors ban hardcoded credentials (`eslint.config.js:120-149`).
- **Backend:** `tgmdp_backend/build.gradle:8` applies `checkstyle` (config `config/checkstyle/checkstyle.xml`, `ignoreFailures = false` at `:310-313`), plus `pmd` (`:9`, rules in `config/pmd/ruleset.xml`) and `spotbugs` (`:10`). `-Pfast` disables Checkstyle and PMD (`:13-16`) for quick local loops. Spotless is present but commented out (`:5`, `:72`). The `.githooks/pre-commit` runs a secret scan then `./gradlew -Pfast test`.
- **Why `-Pfast`:** full static analysis is slow; the team trades depth for speed locally and leaves the full run to CI.
- AMS app: `tgmdp-app-ams/eslint.config.js` plus `package.json:12` (`eslint .`) and `tsconfig.eslint.json`.

## Check yourself
Why does the RN repo repeat in CI the same checks that Husky runs locally?

## Related
- [[Unit Testing]] · [[Test Coverage Gates]] · [[Code Review]] · [[Clean Code Naming]] · [[Web Console Build Lint and Testing]] · [[Vendor Portal Build Lint and Testing]] · [[Best Practices MOC]]
