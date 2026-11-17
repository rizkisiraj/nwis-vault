---
tags: [codebase, tgmdp-vendor-frontend]
---
# Vendor Portal Build, Lint and Testing

Up: [[tgmdp_vendor_frontend]] · [[System Overview]]

The vendor portal's toolchain is the same family as the admin console's (Vite 7, TypeScript 5.9, ESLint 9, Vitest 4), but configured more lightly, with Prettier added for formatting.

## Scripts

From `package.json`:

| Script | Runs | Note |
|---|---|---|
| `dev` / `dev:expose` | `vite` (`--host` to reach it from a phone on the LAN) | |
| `build` | `vite build`, with `prebuild: tsc -b` | npm runs `prebuild` automatically first, so type errors still fail the build |
| `lint` / `lint:fix` | `eslint .` | |
| `format` / `format:check` | `prettier --write .` / `--check` | not present in the console |
| `test`, `test:run`, `test:watch`, `test:ui`, `test:related` | Vitest | |
| `test:coverage` | `vitest run --coverage` | see the caveat below |
| `scaffold:feature <name>` | `scripts/scaffold-feature.js` | generates a feature skeleton |
| `deploy:dev` / `deploy:prod` | docker-compose | |

`tsc -b` ("build mode") follows project references in `tsconfig.json`, type-checking `tsconfig.app.json` and `tsconfig.node.json` with incremental build info. The console uses plain `tsc && vite build` in one script; the effect is similar.

## Vite

```ts
// vite.config.ts:10-26 (trimmed)
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), "");
  return {
    base: env.VITE_BASE_PATH || "/",
    plugins: [react(), tailwindcss()],
    resolve: { alias: { "@": path.resolve(__dirname, "src") } },
    server: { allowedHosts: env.VITE_ALLOWED_HOSTS ? env.VITE_ALLOWED_HOSTS.split(",").map((h) => h.trim()) : [] },
  };
});
```

Same shape as the console's: `loadEnv` for config-time variables, `base` for sub-path hosting, the `@/` alias. Two differences: no `vite-tsconfig-paths` plugin (the alias is declared directly), and no PostCSS/SCSS pipeline, only the Tailwind v4 Vite plugin. The router's `import()` calls make Vite emit one chunk per page automatically; no extra config is needed for [[Single Page Applications|code splitting]]. See [[Build Tools and Dependency Management]].

As in the console, every `VITE_*` value is inlined into the public bundle at build time. Nothing secret belongs there. See [[Secrets Management]].

## TypeScript

`tsconfig.app.json` matches the console's strictness: `strict`, `noUnusedLocals`, `noUnusedParameters`, `verbatimModuleSyntax`, `erasableSyntaxOnly`, `noFallthroughCasesInSwitch`, `noUncheckedSideEffectImports`. It adds `"baseUrl": "."` for the `@/*` path mapping and includes test files explicitly. See [[Type Systems]].

## ESLint and Prettier

The ESLint config is short:

```js
// eslint.config.js:9-22
export default defineConfig([
  globalIgnores(["dist"]),
  {
    files: ["**/*.{ts,tsx}"],
    extends: [
      js.configs.recommended,
      tseslint.configs.recommended,
      reactHooks.configs.flat.recommended,
      reactRefresh.configs.vite,
    ],
    languageOptions: { ecmaVersion: 2020, globals: globals.browser },
  },
]);
```

Only the recommended presets, no house rules. Style is delegated to Prettier (`.prettierrc`):

- `@trivago/prettier-plugin-sort-imports` sorts imports into groups: React first, then scoped packages, other packages, `@/` aliases, relative imports, then CSS. You can see the result at the top of every file (blank lines between groups).
- `prettier-plugin-tailwindcss` sorts Tailwind class names into a canonical order, so diffs do not churn when two people order classes differently.
- `singleAttributePerLine: true` puts each JSX prop on its own line, again for cleaner diffs.

This split, "linter for bugs, formatter for style", is the common modern setup. The console takes the other route and encodes style in ESLint rules (`semi`, `comma-dangle`, `object-curly-spacing`). See [[Linting and Formatting]].

## Vitest

```ts
// vitest.config.ts:6-17
export default defineConfig({
  plugins: [react()],
  test: { globals: true, environment: "jsdom", setupFiles: ["./vitest.setup.ts"] },
  resolve: { alias: { "@": path.resolve(__dirname, "./src") } },
});
```

`vitest.setup.ts` adds jest-dom matchers and replaces `localStorage` with an in-memory object, because Zustand `persist` writes to it on every `set()`.

Unlike the console, the setup does **not** force mock mode. Tests that need isolation mock the module explicitly:

```ts
// src/features/TGMDPVENc003/store/__tests__/useTGMDPVENc003Store.test.ts:7-12
vi.mock("../../api/TGMDPVENc003", () => ({
  TGMDPVENc003Api: { search: vi.fn(), getPermissions: vi.fn() },
}));
```

That is arguably cleaner: each test states its own dependencies instead of relying on a global env flag. See [[Mocking]] and [[Unit Testing]].

### What is tested

81 test files. Every `TGMDPVEN*` feature has some, concentrated on:

- **stores**: state transitions such as "clears previous results before a new search",
- **pages**: user flows with Testing Library (`TGMDPVENb001/pages/__tests__/` alone has 11 files),
- **components**: forms and tables,
- **infrastructure**: `src/lib/__tests__/client.test.ts` exercises the interceptors directly, including Blob-to-JSON error recovery. It reaches into `client.interceptors.response.handlers` to call the rejected handler with a fake `AxiosError`, which is a pragmatic way to unit-test an interceptor without a network. `src/components/__tests__/ProtectedRoute.test.tsx` covers the session guard.

### Coverage caveat

`npm run test:coverage` asks Vitest for coverage, but neither `@vitest/coverage-v8` nor `@vitest/coverage-istanbul` is in `package.json`, and no `coverage` block exists in `vitest.config.ts`. In this checkout the provider package is not installed under `node_modules/@vitest/`, so the command will likely stop and ask to install it. Coverage is therefore not measured by default here. The console has the provider configured and enabled on every run. See [[Test Coverage Gates]].

There are no Playwright or other end-to-end tests in this repo.

## CI and containers

- There is no `.gitlab-ci.yml` in the repo, so lint, format, type-check and tests are not enforced by a pipeline from this repository. They depend on developer discipline and code review ([[Code Review]]).
- `docker/Dockerfile` + `docker/nginx.conf` follow the same multi-stage dev / build / nginx pattern as the console. `.dockerignore` keeps `node_modules` and `dist` out of the build context.

## Try it yourself

1. Run `npm run format:check`. If it reports files, run `npm run format` and look at the diff: how much is import order, how much is class order?
2. Install `@vitest/coverage-v8` locally (do not commit), run `npm run test:coverage`, and find the least-covered feature.
3. Add `"no-console": "warn"` to `eslint.config.js` and count warnings. The 401 handler in `lib/client.ts` uses `console.error`; is that one acceptable?

## Related

- Up: [[tgmdp_vendor_frontend]] · [[System Overview]]
- Siblings: [[Vendor Portal Folder and Feature Structure]] · [[Vendor Portal API Layer]] · [[Vendor Portal State and Forms]]
- Compare: [[Web Console Build Lint and Testing]] · [[Web Frontends Compared]]
- Concepts: [[Build Tools and Dependency Management]] · [[Type Systems]] · [[Testing Pyramid]] · [[Mocking]] · [[Configuration and Environments]] · [[Single Page Applications]]
- Practices: [[Linting and Formatting]] · [[Unit Testing]] · [[Test Coverage Gates]] · [[Code Review]] · [[Secrets Management]]
