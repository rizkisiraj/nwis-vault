---
tags: [codebase, tgmdp-web-frontend]
---
# Web Console Build, Lint and Testing

Up: [[tgmdp_web_frontend]] · [[System Overview]]

This note covers the toolchain: how code is compiled and bundled, how style rules are enforced, and how the ~660 test files are run.

## Scripts you will actually use

From `package.json`:

| Script | What it runs | When |
|---|---|---|
| `npm run dev` | `vite` dev server with HMR on :5173 | daily work |
| `npm run build` | `tsc && vite build` | before a release; fails on any type error |
| `npm run lint` / `lint:fix` | `eslint .` | before commit |
| `npm run test` | `vitest` in watch mode | while coding |
| `npm run test:run` | one pass, exits | CI-style check |
| `npm run test:coverage` | `vitest run --coverage` | checking coverage |
| `npm run test:related` | only tests affected by changed files | fast pre-commit |
| `npm run e2e` | Playwright against a running app | integration checks |
| `npm run deploy:dev` / `deploy:prod` | docker-compose dev server / nginx image | containers |

## Vite: dev server and bundler

```ts
// vite.config.ts:10-31 (trimmed)
export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), "");
  return {
    base: env.VITE_BASE_PATH || "/",
    server: { allowedHosts: env.VITE_ALLOWED_HOSTS ? env.VITE_ALLOWED_HOSTS.split(",").map((h) => h.trim()) : [] },
    plugins: [react(), tsconfigPaths(), tailwindcss()],
    resolve: { alias: { "@": path.resolve(__dirname, "./src") } },
  };
});
```

- In development Vite serves native ES modules and transforms files on request, so start-up is near-instant. For production it runs Rollup to produce hashed, minified bundles in `dist/`. See [[Build Tools and Dependency Management]].
- `loadEnv` is needed because `vite.config.ts` runs in Node, where `import.meta.env` is not populated. The comment on lines 7-9 explains this.
- `base` must match the router `basename`, or asset URLs break when the app is served under a sub-path.
- Only variables prefixed `VITE_` reach browser code, and they are **baked into the bundle at build time**. Anything in a `VITE_` variable is public. Never put a secret there. See [[Secrets Management]] and [[Configuration and Environments]].
- `@/` is an alias for `src/`, so imports read `@/shared/api/client` instead of `../../../../shared/api/client`.

Note: the routes in `src/app/router/routes.tsx` import pages eagerly, so the whole console ships as one large bundle. The vendor portal uses lazy routes instead ([[Web Frontends Compared]]).

## TypeScript configuration

`tsconfig.json` is a solution file that references `tsconfig.app.json` (browser code) and `tsconfig.node.json` (config files). Key options in `tsconfig.app.json`:

- `"strict": true` turns on `strictNullChecks`, `noImplicitAny` and friends.
- `"noUnusedLocals"`, `"noUnusedParameters"`: dead code is a compile error. This is why unused callback parameters are prefixed `_` (`setFilter: (_filter) => void`).
- `"verbatimModuleSyntax": true` forces `import type { X }` for type-only imports, so the bundler can drop them safely.
- `"erasableSyntaxOnly": true` bans TS features that emit runtime code (enums, namespaces, parameter properties). Types must be erasable, keeping the output plain JavaScript.
- `"noEmit": true`: `tsc` only type-checks. Vite (esbuild) does the actual transpiling.

See [[Type Systems]].

## ESLint

`eslint.config.js` uses the flat config format. It layers:

1. `js.configs.recommended` and `typescript-eslint` recommended rules.
2. `eslint-plugin-react-hooks` (rules of hooks, exhaustive deps) and `react-refresh` (keeps Fast Refresh working by warning when a file exports non-components).
3. House rules (lines 85-131), including `eqeqeq`, `prefer-const`, `no-var`, `camelcase`, `semi: always`, `curly: multi-line`, `brace-style: 1tbs`, `no-console: warn`, `max-len: 300`, `comma-dangle: always-multiline`.

The comments in the file are in Chinese and state the purpose of each plugin. The team follows a guideline PDF (mentioned on lines 77-80) and avoids turning rules off without reason. When code does disable a rule, it is local and explicit, for example `// eslint-disable-next-line react-hooks/exhaustive-deps` on a run-once effect (`AppLoginLogoutRecordQueryPage.tsx:75`). See [[Linting and Formatting]].

There is no Prettier in this repo. Formatting is enforced only through the ESLint style rules above.

## Vitest

```ts
// vitest.config.ts:12-25
test: {
  coverage: {
    enabled: true,
    provider: "v8",
    reporter: ['text', 'html'],
    exclude: ['src/features/**/api/**', 'src/features/**/types/**', 'src/features/**/routes.tsx'],
  },
  globals: true,
  environment: 'jsdom',
  setupFiles: './vitest.setup.ts',
},
```

- `environment: 'jsdom'` gives tests a fake DOM.
- Coverage is on for every run and uses V8's built-in instrumentation. `api/`, `types/` and `routes.tsx` are excluded: api files are thin HTTP wrappers (tested indirectly through mocks), types have no runtime code, and route tables are declarations. No numeric threshold is configured here, so coverage is reported, not enforced by the tool. Team process (the workspace's DoD skills) asks for high coverage on changed modules instead. See [[Test Coverage Gates]].

### The setup file explains real-world friction

`vitest.setup.ts` is worth reading because each block fixes a concrete problem:

- `vi.stubEnv("VITE_USE_MOCK", "true")`: **every test runs in mock mode**. Services resolve to the `.mock.ts` implementations, so no test touches the network.
- `fake-indexeddb/auto`: Dexie (IndexedDB) needs a backend in jsdom.
- `ResizeObserverMock`: AntD's `Watermark` uses `ResizeObserver`, which jsdom lacks.
- `vi.mock("/images/...")`: public-folder images imported by absolute path resolve in Vite but not in Vitest.
- `MemoryStorage`: in this environment `localStorage` arrived without `setItem`, which crashed every `persist` store. The setup installs a Map-backed `Storage` only if the real one is broken.

### Test types you will find

| Level | Example | What it proves |
|---|---|---|
| Schema test | `schema/__tests__/appLoginLogoutRecordQueryScheme.test.ts` | validation and transform rules |
| Store test | `store/__tests__/useAppLoginLogoutRecordQueryStore.test.ts` | state transitions, error handling |
| Page test | `pages/__tests__/AppLoginLogoutRecordQueryPage.test.tsx` | rendering + user interaction with Testing Library |
| Guard test | `features/auth/components/guards/__tests__/TaskRouteGuard.test.tsx` | permission rules |
| Util test | `features/auth/utils/__tests__/sys-auth.test.ts` | ported legacy logic |

A store test isolates the store from the network by mocking the service module:

```ts
// store/__tests__/useAppLoginLogoutRecordQueryStore.test.ts:7-13
vi.mock("../../api", () => ({
  appLoginLogoutRecordQueryService: {
    getRecords: vi.fn(),
    getInit: vi.fn(),
    exportRecords: vi.fn(),
  },
}));
```

Because the store imports from `../api`, this one `vi.mock` replaces the dependency. That is the payoff of the one-way `page → store → api` layering. See [[Mocking]] and [[Unit Testing]].

Page tests render with `src/test/test-utils.tsx`, a custom `render` that wraps the UI in a `MemoryRouter`, so components that call `useNavigate` work without a browser. Testing Library encourages querying by visible text and role, which tests behaviour instead of implementation details.

## Playwright (end-to-end)

`playwright.config.ts` targets a real deployed app (`VITE_APP_URL`) with real accounts from `.env`. It runs **serially** (`fullyParallel: false`) because the specs share one backend account and mutable data; parallel runs would overwrite each other's writes (comment on lines 35-40). The configured `testDir` is `./src/test/integration`, which does not exist in the current checkout, so E2E specs appear to be kept elsewhere or not yet committed. Treat this as unverified. Together these levels form the [[Testing Pyramid]]: many fast unit tests, fewer slow E2E tests.

## CI and containers

- `.gitlab-ci.yml` only includes `.autoqa/analyze.gitlab-ci.yml`, a read-only job that asks an AI to propose test cases. **There is no lint/test/build gate in this repo's CI file.** Quality gates are run by developers and by the workspace's review skills. That is a gap worth knowing.
- `docker/Dockerfile` is multi-stage: `development` runs the Vite server, `build` runs `npm run build`, `production` copies `dist/` into `nginx:alpine`.
- `openspec/` holds spec-driven change proposals used with AI coding assistants (`openspec/config.yaml` injects project context into each proposal).

## Try it yourself

1. Run `npm run test:run -- src/features/record-query/app-login-logout-record-query` and read the coverage table for that folder.
2. Delete the `ResizeObserverMock` block locally and run a page test that renders `AppLayout`. What error do you get?
3. Add `"noUncheckedIndexedAccess": true` to `tsconfig.app.json` and run `npx tsc --noEmit`. Count the errors and pick one to fix properly.

## Related

- Up: [[tgmdp_web_frontend]] · [[System Overview]]
- Siblings: [[Web Console Folder and Feature Structure]] · [[Web Console API Layer]] · [[Web Console State and Forms]]
- Compare: [[Vendor Portal Build Lint and Testing]] · [[Web Frontends Compared]]
- Concepts: [[Build Tools and Dependency Management]] · [[Type Systems]] · [[Testing Pyramid]] · [[Mocking]] · [[Configuration and Environments]]
- Practices: [[Linting and Formatting]] · [[Unit Testing]] · [[Test Coverage Gates]] · [[Code Review]] · [[Secrets Management]] · [[Environment Separation]]
