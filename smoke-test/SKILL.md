---
name: smoke-test
description: Standards for writing, reviewing, or modifying smoke tests in the inventory-manager repo. Covers test classification, self-contained per-run fixtures (no permanent-seed dependence), hybrid real-API provisioning with an admin-SQL fallback, hard-delete teardown, domain-event waits over sleeps, strict entity-specific assertions, 5xx-vs-4xx failure classification, and the session sweep as a crash backstop. Use when creating or editing a web UI smoke (libs/webui-smoke, a real browser against the deployed dev app) or a Python Lambda smoke (libs/lambda-smoke), reviewing a smoke-test PR, or when the user mentions smoke tests, flaky fixtures, seed prerequisites, or smoke cleanup/sweep.
---

# Smoke Test

A smoke test is not cookie-cutter automation. It is a small, deliberate proof
that a critical deployed or user-facing path works end to end. A good smoke test
is **narrow, deterministic, and diagnostic**: it answers one question, *does this
critical path work right now through the real integration points this product
depends on?*, and when it fails the failure points at a real product, seed,
deployment, or environment problem, never at shared state, a wrong sleep, an
accidental first-row click, or leaked fixtures.

That diagnostic value is the whole point. A clean, self-contained smoke is a
precision instrument: once it stops being flaky, a red run is trustworthy, so it
catches real defects instead of crying wolf. The lessons below were paid for in
[#483](https://github.com/craigtrim/inventory-manager/issues/483); honor them so
they do not have to be relearned.

The two suites in this repo:

- **Web UI smoke** with `cd libs/webui-smoke && make smoke`. A real browser
  (Python Playwright) signed in with real Cognito creds, driving the deployed dev
  CloudFront app through `DevWebSession`. Tests are `<domain>/test_<noun>.py`.
  Replaced the old apps/web Playwright suite, which booted a local `next dev` and
  tested a server that does not exist in production (#552, #592).
- **Lambda smoke** with `cd libs/lambda-smoke && make smoke`. Real HTTP against
  the deployed dev API Gateway. Tests are `<domain>/test_<noun>.py`.

Both run against the deployed dev environment, and together they are the
post-deploy gate via `infra/cdk/smoke-dev.sh`.

## Workflow: writing or reviewing a smoke

1. **Classify the test first.** The fixture strategy and assertion style depend
   on the class. Pick exactly one; do not mix classes in one scenario:
   - `Shell/render smoke`: page/bundle renders its essential regions. Broad
     shape assertions OK.
   - `Read-only seeded contract smoke`: known data is readable and correctly
     shaped. The contract must be explicit and checked up front.
   - `Mutating workflow smoke`: creates/changes data, verifies the exact state
     transition. Per-run fixtures.
   - `Destructive/cleanup smoke`: proves delete/close/cancel. Keep separate from
     ordinary cleanup helpers.
   - `External/auth smoke`: proves Cognito or another external dependency. Keep
     isolated, so external throttling never makes product smoke look broken.

2. **Every test stands alone.** No test may depend on another test, on execution
   order, on prior-run residue, or on mutable permanent seed. A row that a test
   applies, deletes, or completes is consumed on the first run, so a test that
   leans on permanent seed fails on every run after. Provision the data you
   assert on, assert only on that owned data (or the immutable tenant anchors
   `org` / `folder` / `user`, used as scope only), and tear it down. The session
   sweep is a crash backstop, never a thing a test relies on to pass. See
   [REFERENCE.md](REFERENCE.md#self-containment-and-the-session-sweep).

3. **Own your fixtures, with a per-run namespace.** Generate unique
   names/keys/numbers with a recognizable smoke prefix that carries a per-run id
   (e.g. `smoke-<run_id>-...`), store the returned IDs, and assert against those
   IDs, not row order. Fixtures must be *valid* data: build them with the same
   library the product parses with (a hand-pasted blob with the right magic bytes
   but no decodable body is silently skipped, not tested). See
   [REFERENCE.md](REFERENCE.md#fixture-rules).

4. **Behavior runs through the real API; SQL is a setup fallback only.** The
   behavior under test always goes through the real product endpoint. Only setup
   the public API has no route for may fall back to the privileged
   `/admin/migrate` SQL path, and each such fallback carries a `TODO` naming the
   missing endpoint so the gap stays visible. See
   [REFERENCE.md](REFERENCE.md#provisioning-and-the-admin-migrate-fallback).

5. **Tear down hard, in foreign-key order.** The product's own delete routes
   often soft-delete, which leaves residue. Per-test teardown hard-deletes
   (through the admin path) anything whose delete is soft or absent, clearing
   children and items before the rows that RESTRICT-reference them. Add an
   explicit seed-prerequisite check with a crisp message; never let a deep
   downstream `404` be the first signal. See
   [REFERENCE.md](REFERENCE.md#fixture-rules).

6. **Wait on domain events, not sleeps.** Wait on the exact save request/
   response, read back through the canonical endpoint, or wait for the committed
   value in the UI, and wait on the *final* state, not a proxy for it (a vendor
   reaching its full pooled count, not merely appearing). Avoid `waitForTimeout`
   unless there is no observable event. See [REFERENCE.md](REFERENCE.md#waiting-rules).

7. **Assert strictly where the test owns the behavior.** Exact status code,
   exact entity ID/route/name, exact persisted field value, exact activity
   context. Justify any `first()`/`last()`/`nth()`, broad `getByText`, bare
   `toBeVisible()`, `>=` count in a mutation, or multiple accepted status codes
   for one happy path. See [REFERENCE.md](REFERENCE.md#assertion-rules).

8. **Classify failures by HTTP status class.** In the Lambda suite, assert
   status through the `lambda_smoke.checks` helpers (`expect`, `expect_ok`) so a
   failure leads with its class: a 5xx is a server/infra fault (a cold start or a
   deploy/migration gap, worth a re-run to confirm), a 4xx is a real contract or
   validation defect. Transport errors keep their native httpx type, which
   already names them as network faults. The helper is a thin, stateless function
   each test calls, not a hidden harness. See
   [REFERENCE.md](REFERENCE.md#failure-classification).

9. **Run the Pre-PR checklist** in [REFERENCE.md](REFERENCE.md#pre-pr-checklist),
   and note in the PR whether a DEV deploy or a migration must be applied before
   the smoke can pass. Migrations are manual and per-environment; code shipping
   in a PR does not mean its migration ran, so a smoke can legitimately go red on
   an unapplied migration.

## Reject these patterns

Depending on permanent fixed-UUID seed · mutating a seed row and calling the
drift "re-runnable" · accepting `200 or 409` because a prior run changed the
fixture · passing because "the row was already complete from a prior run" ·
asserting transient UI state instead of persisted domain state · sleep-and-hope ·
clicking the first row when the test created a specific one · using cleanup
tolerance to mask product behavior · broadening accepted status codes to hide a
deployed backend bug · mixing setup calls, UI behavior, and unrelated dashboard
assertions in one scenario · failures that are ambiguous between missing seed,
stale deploy, cleanup leak, and product regression.

Suite-specific detail: [REFERENCE.md, Web UI](REFERENCE.md#web-ui-smoke-tests-webui-smoke)
and [Lambda](REFERENCE.md#lambda-smoke-tests).
