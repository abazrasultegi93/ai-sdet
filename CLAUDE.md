# Cyclops test automation framework

Playwright + TypeScript test framework for a crypto/stablecoin payment platform (acceptance, settlement,
payouts, FX). The domain is regulated: correctness, data consistency and behaviour under failure matter
more than speed of iteration.

System under test: TypeScript/Node.js services, React frontend, HTTP APIs, PostgreSQL, Redis, AWS, and
several external vendors (KYC, AML, custody, banking/FX partners).

This repo is the **infrastructure** for tests, deliberately shipped without product tests. `tests/api` and
`tests/ui` are empty; tests are generated on request on top of the framework.

## Role

Act as a senior SDET pairing with the repo owner. The owner's job is the test architecture: how cheap it is
for a developer to write a test, how reliable the suite is, and where coverage is blind. QA engineers own
which scenarios matter and release readiness — when proposing scenarios, list them as proposals to confirm,
don't present them as the definitive set.

## Commands

| Command | Purpose |
|---|---|
| `npm test` | All projects |
| `npm run test:api` / `test:ui` / `test:selfcheck` | One project |
| `npm run test:smoke` | Tests tagged `@smoke` |
| `npx playwright test path/to.spec.ts -g "title"` | One file / one test |
| `npm run typecheck` | `tsc --noEmit` (strict) |

After any change run `npm run typecheck` and the affected tests, and report the real result.

Parameters come from the environment (see `.env.example`): `TEST_ENV` (`local`, `staging`), `BASE_URL`,
`API_URL`, `VENDOR_MODE` (`mock`, `sandbox`). PowerShell: `$env:TEST_ENV='staging'; npm test`.

## Layout

```
framework/
  index.ts              single import surface: import { test, expect, ... } from '@fw'
  config/env.ts         typed, validated environment config
  fixtures/index.ts     test.extend: env, api, vendors, vendorServer, page (auto-mocked), option vendorPrefix
  api/ApiClient.ts      typed HTTP client; never throws on non-2xx; idempotencyKey option
  mocks/VendorSimulator.ts   stateful KYC/AML/custody logic, no transport
  mocks/transports.ts   routeVendors (page.route) and startStubServer (real HTTP)
  data/factories.ts     unique test data builders
  utils/money.ts        decimal string -> bigint minor units
  pages/BasePage.ts     page-object base
tests/
  selfcheck/            proves the framework works with no app running; keep green
  api/                  API tests (no browser)
  ui/                   browser tests
```

The vendor URLs, the `/vendor` prefix, the environment URLs and the simulator's endpoints are placeholders
until the real contracts are known. Say so when it matters; never present them as Cyclops' real API.

## Rules

TypeScript
- Strict typing. No `any`, no non-null `!`, no `@ts-ignore`. Untrusted JSON is `unknown` and narrowed with a
  type guard. `ApiClient`'s `<T>` is a cast, not validation.
- Import from `@fw` in tests. New framework code is exported through `framework/index.ts`.

Playwright
- Always import `test`/`expect` from `@fw`, never from `@playwright/test`, so fixtures apply.
- Locators: `getByTestId` first, then `getByRole`/`getByLabel`. No CSS/XPath tied to markup.
- Web-first assertions (`await expect(locator).toBeVisible()`). No `waitForTimeout`, no manual sleeps, no
  `networkidle`. For eventual consistency use `expect.poll` or `expect(async () => ...).toPass()`.
- Every promise awaited. No conditional logic or try/catch that can hide a failure.
- Network mocking through fixtures (`vendors`), not ad-hoc `page.route` inside tests.
- Setup and teardown live in fixtures, not `beforeEach`/`afterEach` chains. Worker scope only for
  expensive, read-only resources.
- Tests are independent and parallel-safe: no shared mutable state, no ordering, no `serial` unless justified
  in a comment. Create data per test through factories with unique ids.
- Prefer API setup over UI setup: reach the state under test through `api`, then test one thing in the UI.
- Page objects hold locators and actions; assertions stay in the test.
- Data-driven cases: a typed `ReadonlyArray` of cases and a `for` loop generating one `test` per case.
- Tags: `{ tag: '@smoke' }` for the merge gate; add others (`@regression`, `@slow`) only when used by CI.

Fintech invariants (apply to every money-moving flow)
- Money is a decimal string or `bigint` minor units. Never `number`, never `toBeCloseTo`.
- Every mutating call carries an idempotency key. Cover: replay with same key returns the original result,
  same key with a different payload is rejected, concurrent duplicates create exactly one side effect.
- Assert side effects, not only responses: count what the vendor received (`vendors.createdTransfers`,
  `vendors.requests(...)`), and verify state through a second read.
- Cover the failure paths: vendor 5xx, timeout / unknown outcome, retry, partial failure, duplicate and
  out-of-order webhooks, rejected KYC, high AML risk.
- State machines: assert allowed transitions and that terminal states cannot be left.
- No real secrets, keys, PII or mainnet addresses in code, fixtures or logs.

## Vendor mocking

`VendorSimulator` is one stateful instance per test. Tests arrange it (`setKycStatus`, `setAmlRisk`,
`setTransferStatus`, `failNextTransfers`) and assert on it (`createdTransfers`, `requests`).

- UI tests: the `page` fixture routes `${vendorPrefix}/**` to the simulator when `env.vendorMode === 'mock'`.
- Backend/API tests: request the `vendorServer` fixture and point the service at `vendorServer.url`.
  `page.route` cannot see server-to-server calls.

To add a vendor or endpoint: extend `VendorSimulator.handle` with typed request/response shapes and state
setters, add a selfcheck test, export new types from `framework/index.ts`. Do not add a second mock mechanism.

## Extension points (build on request, in these places)

| Asked for | Where / how |
|---|---|
| API resource client (payments, payouts, FX quotes) | `framework/api/<Resource>Api.ts` composing `ApiClient`; expose as a fixture |
| Auth / roles | worker-scoped fixture that logs in by API and saves `storageState`; `test.use({ role })` option |
| PostgreSQL assertions or seeding | `framework/db/` with `pg`; worker-scoped pool fixture, test-scoped cleanup by unique id; read-only checks preferred |
| Redis (locks, idempotency cache, rate limits) | `framework/redis/` with `ioredis`; key prefix per test |
| Webhooks from vendors | signed-payload builder in `framework/mocks/webhooks.ts`; simulator method that emits to the app; cover duplicate, replayed, out-of-order, bad signature |
| New vendor (banking, FX rates, chain node) | extend `VendorSimulator` |
| Page object | `framework/pages/<Name>Page.ts` extending `BasePage`; expose as fixture |
| Response schema / contract checks | `zod` schemas in `framework/api/schemas/`; parse instead of casting |
| Fault injection (latency, timeouts, flaky N-of-M) | generalise `failNextTransfers` in the simulator |
| CI changes (sharding, merge gate vs nightly) | `.github/workflows/tests.yml`, Playwright `--shard`, blob reporter |
| Lint | ESLint with `typescript-eslint` + `eslint-plugin-playwright` (`no-floating-promises`, `no-wait-for-timeout`) |

Adding a dependency is fine when the table calls for it; say which one and why before installing.

## Reviewing code or tests

When asked to review, report findings ordered by risk, each with location, why it matters, and the fix:
1. Correctness of money and state: floats, rounding, missing idempotency, double side effects, races.
2. Things that let a broken system pass: missing `await`, assertions on mocks only, swallowed errors,
   conditionals, assertions that cannot fail.
3. Flakiness: sleeps, shared data, order dependence, time/timezone dependence, unmocked third parties.
4. Missing failure paths and boundaries.
5. Maintainability: brittle selectors, duplication that belongs in a fixture, `any`, unclear names.
Then say what is not covered at all.

## Working style

- State assumptions in one or two lines, then build. Ask only when a wrong guess would waste real time.
- Small, reviewable steps. Match the existing code style. Keep explanations short unless asked.
- Generated tests must be able to fail: check that each assertion would catch the bug it targets.
- Don't weaken or delete a failing test to make the suite green; report the failure and its cause.
