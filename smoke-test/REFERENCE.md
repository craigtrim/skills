# Smoke Test Reference

Full rule detail behind [SKILL.md](SKILL.md). Read the relevant section when
writing or reviewing a smoke test of that class.

## Fixture Rules

Every test owns the fixtures it asserts on.

- Generate unique names, keys, or numbers with a recognizable smoke prefix that
  carries a per-run id (for example `smoke-<run_id>-<label>`), so two overlapping
  runs (local plus CI) never collide and the sweep can target exactly one run.
- Store returned IDs and assert against those IDs, not incidental row order.
- Fixtures must be valid data, not just plausible bytes. Build them with the same
  library the product parses with. A hand-pasted blob with correct magic bytes
  but no decodable body is silently skipped by the code under test, so the test
  proves nothing while looking green.
- Delete or close what the test creates. Hard-delete when the product's own
  delete route soft-deletes (sets `deleted_at`) or has no endpoint, otherwise a
  "successful" run still leaves residue. Hard-delete in foreign-key order: clear
  children and items before the rows that RESTRICT-reference them.
- Keep cleanup idempotent, but do not let cleanup become the only proof that
  delete behavior works.
- If cleanup tolerates `404`, it should still fail on unexpected `401`, `403`,
  `409`, or `500`.

Use fixed DEV seed data only when the test is read-only and treats the data as an
immutable anchor. The seeded `org` / `folder` / `user` are the only such anchors,
and only as scope or destination, never asserted on as proof of broader seeded
state.

Avoid these patterns:

- depending on permanent fixed-UUID seed for anything the test asserts on
- mutating a fixed seed row and calling the resulting drift "re-runnable"
- accepting `200 or 409` because a prior run changed the fixture
- depending on large headroom in a seeded counter or quantity
- using a broad `>=` assertion for a transaction that should have exact effects

If a smoke test depends on seed data, add an explicit prerequisite check with a
clear failure message that names the missing seed and migration. Do not let a
deep downstream `404` be the first signal.

## Self-containment And The Session Sweep

Every test must pass on its own per-test setup and teardown. No test depends on
another test, on execution order, on prior-run residue, or on mutable permanent
seed. The session sweep is a crash backstop, not a load-bearing part of any
test.

The Lambda suite's lifecycle (in `libs/lambda-smoke/conftest.py` and
`lambda_smoke.namespace`):

- **Session start: age-gated sweep.** Reap leftover `smoke-` fixtures from a
  prior or crashed run, but only those older than a stale window, so a run
  executing concurrently is never reaped out from under itself.
- **Per test: own setup and `finally` teardown.** Each test creates its
  namespaced fixtures and deletes them, whatever the outcome.
- **Session end: forced clean of this run's namespace.** Deletes exactly this
  run's `smoke-<run_id>-` rows, regardless of pass or fail.

A hard process kill (SIGKILL, OOM) skips the per-test and end-of-session
cleanup; the next run's start sweep is what makes "clean no matter what" true
across runs. That is the only job the sweep has. If a test cannot pass without
the sweep having run, the test is wrong.

`lambda_smoke.namespace` provides `smoke_name()` / `smoke_key()` for the
per-run namespace and `purge_ids()` for hard teardown of entities with no API
delete; `scripts/sweep-smoke-fixtures.sh` is the deploy-time reaper.

## Provisioning And The Admin-Migrate Fallback

The behavior under test always runs through the real product API. Provisioning
the setup state may use a fallback only where the public API genuinely has no
route for it.

- Drive the action under test (the apply, the fulfil, the receipt, the cascade
  delete) through its real endpoint. Never simulate it in SQL.
- For setup the API cannot express (creating a pick list, a work order, a PO
  line, a precise stock-count state, report snapshots/movements), the Lambda
  suite POSTs SQL to the privileged `/admin/migrate` route via
  `lambda_smoke.admin.run_admin_sql` (runs as the postgres master).
- Mark every such fallback with a `TODO` naming the missing endpoint, so the
  gap stays visible and does not quietly harden into permanent scaffolding.
- SQL-inserted rows still carry the run's `smoke-<run_id>-` namespace so the
  sweep and `purge_ids` can reclaim them.

## Failure Classification

A smoke failure should say what *kind* of failure it is, so a reader can tell a
flaky environment from a real defect at a glance.

- In the Lambda suite, assert status through `lambda_smoke.checks`:
  `expect(resp, *allowed)` (drop-in for `assert resp.status_code == N`) and
  `expect_ok(resp)` (drop-in for `resp.raise_for_status()`). On a mismatch the
  `AssertionError` leads with the class: `HTTP 5xx SERVER/INFRA` (a server error,
  often a cold start or a deploy/migration gap, worth a re-run to confirm) versus
  `HTTP 4xx FUNCTIONAL` (a real contract or validation defect the test caught).
- A 5xx is still a failure. The helper labels it; it never swallows or retries.
- Transport-level faults (connect/read timeouts, TLS handshake timeouts, resets)
  are left as their native httpx exception types, which already name themselves
  as network faults. Do not wrap requests in a shared retry/classify harness; the
  helper is a thin stateless function each test calls, with nothing to test.
- A presigned S3 GET must be fetched with a plain client. If the session injects
  an `Authorization` header, S3 rejects the presigned request with a 400, because
  the URL already carries its auth in the query string.

## Waiting Rules

Do not use sleeps as the primary correctness barrier.

Prefer domain barriers:

- wait for the exact API request and response that represents the save
- read back through the canonical API endpoint and assert persisted state
- wait for the UI to show the committed value after the save resolves
- filter activity by the entity ID and unique value written by the test

Short request-spy windows are acceptable for "no request should fire" tests, but
make the intent explicit. For example, a no-op edit may wait briefly while a
registered request listener proves no matching `POST /items/edit` occurred.

Avoid `waitForTimeout` unless there is no observable event to wait on. If you
must use it, explain why and keep it local to that test.

## Assertion Rules

Assertions should be strict where the behavior is owned by the test.

Good assertions:

- exact status code for the scenario under test
- exact entity ID, route, or item name created by this test
- exact persisted field value after a mutation
- exact activity context for the row written by this test
- exact error message/status for validation and not-found branches

Weak assertions that need justification:

- `first()`, `last()`, or `nth()` selectors
- broad text searches on a whole page
- `toBeVisible()` without verifying the row/entity is the one under test
- `>=` counts in mutating workflows
- multiple accepted status codes for the same happy path

Use row order only when row order is the behavior being tested. Otherwise,
identify the row by fixture ID, unique name, test ID, href, or API response.

## Web UI Smoke Tests (webui-smoke)

The web smoke lives in `libs/webui-smoke`: Python pytest driving a real browser
(Playwright) signed in with real Cognito creds against the deployed dev app via
`DevWebSession`. It replaced the old apps/web Playwright suite, which booted a
local `next dev` (#552, #592). Use it for real browser behavior: routing,
rendering, controls, focus, keyboard behavior, form submission, optimistic UI,
and visible error states.

Guidelines:

- Use the deployed `/api/proxy/*` routes for setup and teardown, naming every
  fixture `smoke-<run_id>-...` (and `smoke_<run_id>_...` for slug keys) so the
  run-scoped sweep reaps it. One session-scoped Cognito sign-in per run.
- Use API calls for setup and teardown.
- Use the UI for the behavior the test claims to cover.
- Avoid direct API mutation in the middle of a UI workflow. If the API behavior
  matters, make it a separate API or Lambda smoke.
- Prefer stable `data-testid`, role, label, href, or fixture ID selectors.
- Avoid broad `getByText` assertions when several rows can contain the same
  text.
- For activity feeds, filter by the item/variant ID and unique change value.
- For save flows, wait on the save response or read-back state, not only on a
  transient "Saved" flash.
- Keep auth-focused tests separate from product-flow tests. Cognito throttling
  should not make unrelated product smoke look broken.

When a web UI test creates data, its cleanup marker must be covered by the
sweeper (per-test teardown is the belt; the conftest session-end sweep and the
external `scripts/sweep-smoke-fixtures.sh` are the suspenders). A new fixture
prefix without sweep coverage is a leak risk.

## Lambda Smoke Tests

Use Lambda smoke tests for real deployed HTTP behavior through API Gateway,
worker Lambda code, RDS Proxy, Postgres, RLS, and Cognito auth.

Guidelines:

- Keep tests real HTTP. Do not mock Lambda smoke.
- Assert exact status codes for the branch under test.
- Validate side effects through the public read endpoint paired with the write.
- Add seed prerequisite checks before seeded-domain tests.
- Use per-run fixtures for mutating workflows whenever possible.
- Do not hide deployed backend bugs by broadening expected status codes.
- If deployed DEV has not yet received the code or seed migration needed by the
  test, report that as a deployment/seed prerequisite, not as a test pass.
- Keep destructive endpoint coverage strict and separate from cleanup helpers.

For transaction smokes, prove the cross-table effect that makes the Lambda
important: updated rows, activity log context, rollback behavior, and terminal
status should be checked explicitly.

## Cleanup And Sweep Rules

Cleanup exists to return the environment to a usable state. It is not a
substitute for behavior coverage.

- Cleanup should be best-effort only where it protects the suite from leaked
  fixtures.
- Unexpected cleanup errors should remain visible.
- A normal suite exit should report leaked current-run fixtures.
- Sweep code must use an allow-list of smoke-owned prefixes, never broad
  delete-all logic.
- A start-of-run sweep must be age-gated so it never reaps a concurrent run's
  in-flight fixtures; it targets only stale (prior/crashed) runs.
- Every new fixture naming pattern must be added to sweep coverage or generated
  through a shared fixture helper that is already swept.
- The sweep is a backstop, not a crutch: a test must pass on its own teardown
  without it. See [Self-containment](#self-containment-and-the-session-sweep).

## Pre-PR Checklist

Before sending a smoke-test PR, confirm:

- The test has one clearly named behavior.
- The test class is clear: shell, read-only seeded, mutating workflow,
  destructive, or auth/external.
- The test is self-contained: it passes on its own setup and teardown, with no
  dependence on another test, on order, on prior-run residue, or on mutable
  permanent seed. It would still pass if the session sweep never ran.
- Mutating tests own their fixtures under a per-run namespace and hard-delete
  them (the product's own delete may only soft-delete).
- Fixtures are valid data, built with the library the product parses with.
- The behavior under test runs through the real API; any `/admin/migrate` SQL is
  setup-only and carries a `TODO` naming the missing endpoint.
- Seed prerequisites fail with crisp messages.
- Assertions identify the specific entity under test; Lambda status checks go
  through `expect` / `expect_ok` so failures classify 5xx vs 4xx.
- Saves and async updates wait on request/response or read-back state, on the
  final value rather than a proxy for it.
- Cleanup is idempotent but does not hide unexpected failures.
- Sweep coverage includes any new fixture prefixes; the start sweep is
  age-gated.
- The PR notes whether a DEV deploy or a (manual, per-environment) migration is
  required before the smoke can pass.
