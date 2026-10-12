# CLAUDE.md

Test automation framework for a crypto/stablecoin payment platform (acceptance, settlement,
payouts, FX). Regulated money movement: priority is **correctness, data consistency, behaviour
under failure**, then everything else.

The repo is infrastructure with no product tests. The task, the system under test and its
contracts arrive at runtime. Nothing below assumes a specific endpoint, vendor or page:
discover it from the input, then adapt.

## Live state (imported, never copied here)

Read these instead of trusting any description of them. They are the source of truth for
commands, parameters and available fixtures.

- Scripts and dependencies: @package.json
- Parameters and their defaults: @.env.example
- Fixtures a test can request: @src/fixtures/index.ts
- Layers (projects) and reporters: @playwright.config.ts

When a script, variable or fixture is added, this file does not change. Edit this file only
when a rule changes.

## Step 0 for every request: orient

1. Classify the request into a mode (table below). If it spans several, run them in order.
2. Identify the inputs you were given: prose, an OpenAPI/JSON sample, source code, a failing
   test, a URL. Derive contracts from them; do not invent fields or endpoints. Anything you had
   to guess is listed under "Assumptions" in the answer.
3. Check what is reachable: is there a running API/UI/DB for `TEST_ENV`? If not, say so and use
   the "no system available" path below rather than pretending tests passed.

| Request looks like | Mode | Output |
|---|---|---|
| "review / what's wrong with / is this good" | **Review** | findings by risk, no rewrite until asked |
| "how would you test / what should we cover" | **Design** | risk analysis + scenario table, then stop |
| "write / add / generate tests" | **Implement** | Design first if no agreed scenarios, then code, then run |
| "failing / flaky / slow" | **Diagnose** | reproduce, root cause, fix, proof |
| "add support for / make it possible to" | **Extend** | framework change + self-check test |
| "why / explain / compare" | **Explain** | short answer with trade-offs |

Slash commands `/test-design`, `/implement-tests`, `/review-tests` are the parameterized
entry points for the first three.

## Modes

**Design.** Restate the behaviour and what failure would cost. Select the risk triggers that
apply (next section). Produce a table: scenario, layer, risk covered, tag, priority. Add what
is deliberately out of scope and which framework pieces are missing. Scenarios are proposals:
QA owns the final list, so stop for confirmation.

**Implement.** Order: contract schema, factory, client method or fixture, then specs. Lowest
layer that can prove the behaviour. Run typecheck and the new specs; report real output. Prove
one test can fail by breaking its expectation once.

**Review.** Findings most severe first, each with location, how it fails or lets a bug through,
and the fix. Order of concern: wrong money or state; tests that cannot fail (missing `await`,
status-only assertions, asserting the mock, swallowed errors, conditionals); flakiness (sleeps,
shared data, order, time); missing failure paths; maintainability. End with what has no test
at all.

**Diagnose.** Reproduce first (`--repeat-each`, `--workers=1` vs parallel, trace). Name the
race or dependency. Never fix by adding retries, sleeps or longer timeouts. A flaky test may be
a real product race: say which one it is.

**Extend.** One mechanism per concern; grow an existing fixture or client before adding a new
one. Every new capability gets a self-check test that runs with no application.

## Risk triggers (apply only those the input activates)

| If the behaviour... | Then cover |
|---|---|
| moves or records value | ledger invariant (debits = credits, fee + net = gross); state verified by a second read or DB row, not only the response |
| is a mutating call | idempotency: replay returns the original, same key + different body rejected, concurrent duplicates create one effect |
| reads then writes a shared resource | concurrency: racing requests, double spend, lost update |
| has statuses | allowed transitions, terminal states cannot be left, out-of-order events |
| involves amounts or rates | precision per asset (6/8/18 decimals), rounding direction, zero/negative/max, dust |
| calls an external party | timeout with unknown outcome, 5xx, malformed body, retry; system ends consistent and retryable |
| receives webhooks | duplicate, replayed, out-of-order, bad signature |
| is asynchronous | poll for the outcome with a timeout; assert no duplicate effect |
| is multi-tenant or role-based | tenant A cannot see or act on tenant B; missing, expired, wrong-scope token |
| is user-facing | one thin UI flow; reach the state through the API first |

## Parameterization: nothing that varies is hardcoded

| Axis | Mechanism |
|---|---|
| Environment / target URLs / credentials | `TEST_ENV` + `src/config/env.ts` (zod schema). New variable: schema + `.env.example` |
| Layer | Playwright project (`--project`) |
| Subset | tags (`@smoke`, `@critical`, `@slow`) + `--grep` |
| Input variations of one rule | typed case table + `for (const c of cases) test(...)` |
| Per-test entities | factories with `uniqueId`; override only the fields the test is about |
| Role, tenant, asset, vendor behaviour | fixture option set with `test.use({ ... })`, default in the fixture |
| Scale | `WORKERS`, `--shard`, `--repeat-each` |

If a test needs a literal URL, id, token, amount precision or role name, that is a missing
parameter: add the axis, do not inline the value.

## No system available

When the task describes a system that is not running locally (common for a spec or a code
snippet), do not stop at design:

1. Encode the described contract as zod schemas.
2. Build a minimal in-process stub (`node:http`, under `src/stubs/`) that implements the
   contract with the state needed for the scenarios, exposed as a fixture that returns its URL.
3. Run the tests against it and state clearly that they prove the tests and the contract
   reading, not the real service.

## Fixed rules

- Import `test`, `expect`, `expectJson` from `@fixtures`, never from `@playwright/test`.
- One behaviour per test; the title states the rule. No branching, loops or try/catch in a
  test body.
- No `waitForTimeout`, sleeps or `networkidle`. Use `expect.poll` / `toPass` with a timeout.
- Money is a decimal string or bigint minor units (`toMinorUnits`). Never `number`,
  `parseFloat`, `toBeCloseTo`.
- Untrusted JSON is `unknown` until parsed by a schema. No `any`, no `@ts-ignore`.
- Tests are independent and parallel-safe; data is created per test.
- UI locators: `getByRole`, `getByLabel`, `getByTestId`.
- No secrets, PII or real addresses in code or logs. Never target production.
- Do not weaken or delete an assertion to get green. Report the failure and its cause.
- New dependency: say which and why before installing.

## Answer format

Assumptions (if any), what was done, real command output, what is not covered, open questions.
Keep it short; the reader is watching live.

## Gotchas

- Playwright joins paths to `baseURL` like a browser: a leading `/` drops the base path prefix.
- `db` and `redis` are lazy, worker-scoped; requesting one opens the connection.
- PowerShell: `$env:TEST_ENV='staging'; npm run test:api`.
