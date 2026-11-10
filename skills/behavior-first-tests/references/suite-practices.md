# Suite-Level Practices

Use when setting up or reviewing infrastructure that keeps a whole suite trustworthy: mutation testing in CI, coverage policy, test sizes, flaky-test handling, test selection, fake ownership, schema drift and snapshot policy. Routine test writing does not need it. Propose infrastructure changes; do not add tools, CI jobs or config without authorization. Evidence and limits are in [research](research.md) sections 11–18. Items marked *not verified* come from inference in the source research, not from fetched tool documentation; confirm them against the installed tool version.

## Mutation testing in CI

- **Diff-scoped and advisory.** Mutate changed, covered lines on each PR and post specific survivors as review comments. Never gate merges on a global mutation score; equivalent mutants distort it.
- **Periodic full runs.** Incremental and diff runs miss indirect effects, so schedule full runs (nightly or weekly).
- **StrykerJS.** The vitest runner always uses per-test coverage, runs single-threaded and does not support browser mode. `--incremental` (or `"incremental": true`) reuses earlier results stored in `reports/stryker-incremental.json`; keep that file as a CI artifact seeded from the main branch, not committed. It does not detect changes to dependencies, environment variables or `.snap` files, so full runs need `--force`. Scope PR runs to changed files with `--mutate`. Whether the vitest runner reports the per-test details incremental mode needs to track test changes is *not verified*.
- **cargo-mutants.** PR runs: `git diff <base> > DIFF && cargo mutants --in-diff DIFF`; the docs call it no substitute for a full run, and test-only diffs produce no mutants. Full runs: `--shard k/n` across a CI matrix with identical arguments on every shard, plus `--baseline=skip` when an earlier job already ran the tests, and an explicit `--timeout`. Cost levers: a `[profile.mutants]` inheriting `test` with `debug = "none"` selected by `profile = "mutants"` in `.cargo/mutants.toml`, skipping doctests (`-- --all-targets`), and a faster linker.
- **Flaky tests corrupt kill signals.** A mutant "killed" by a flaky test is not evidence; deflake or exclude such tests before trusting mutation results.

## Arid skip list

Keep one shared list of code not worth mutating, in the mutation tool configs (Stryker config, `.cargo/mutants.toml`), each entry with a one-line reason. Start from the categories that removed most noise at Google: logging and tracing macros, metrics and telemetry, timeouts/retry/backoff, feature-flag plumbing, collection capacities and debug/display formatting. cargo-mutants supports skipping by source attribute, path filter and function-name regex, and `--list` previews what will run; the exact attribute and config key names (`#[mutants::skip]`, `exclude_re`) are *not verified*. Grow the list from "not useful" review feedback and keep it in the repository, where false positives become the next rule. Do not skip code whose timing or flag behavior is itself a user contract.

## Coverage

- No coverage gate or target, global or per diff. A target becomes a ceiling and rewards tests that only execute lines; half of Meta's useful mutant-killing tests added no line coverage.
- Use coverage while authoring and reviewing to find unexecuted branches, then ask which assertion would tell correct behavior from a plausible defect there.
- Judge suite strength by surviving mutants on changed lines and pseudo-tested functions (a body replaced without any test failing), not by percentages.

## Test sizes

Define tiers by the resources a test may use, not by the label "unit" or "integration":

- **Small:** one process, no network, no real disk or database, no sleeps. Runs on every change.
- **Medium:** local services only (a database or service container on localhost), no external hosts. Runs on every change, in its own job.
- **Large:** the deployed stack end to end. A thin smoke set of critical journeys; edge cases belong in smaller tiers.

Flakiness rises with size, so track flaky rates and set the flake budget per tier, not for the whole suite, and run each tier as a separate CI job so a large-tier flake cannot block small-tier signal. Enforce the limits where the runner allows it (no network in the small tier) rather than by convention. Tier choice does not change the mock policy: DB and external-service doubles stay allowed in any tier.

## Flaky tests

- **Retry to detect, never to hide.** A retry that flips red to green flags the test and opens an issue; it does not silently turn the run green. Runner retry options (vitest `retry`, cargo-nextest retries) can feed detection; their flaky-reporting behavior is *not verified*.
- **Investigate before quarantining.** Missing waits on async work are the leading cause, and a newly flaky test can expose a production race. Read the failing path before suppressing it.
- **Quarantine design.** Quarantine through a checked-in list or annotation linked to an issue with an owner (CODEOWNERS can route it). Quarantined tests keep running in a non-blocking job, return automatically after a run of clean results, and have a size and age cap; quarantined tests protect nothing. Never rewrite results in place, and never let merges depend on a flakiness service being available.
- **Keep history.** Store per-test results (for example JUnit reports) as CI artifacts. Keep flakiness data out of build-cache inputs so caching stays hermetic.
- **Scale.** At thousands of tests, a checked-in list, issue links and a visible flaky table are enough; Bayesian scores and learned selection were built for orders of magnitude more runs.

## Test selection

Select tests from the dependency graph on PRs only with the full suite as a safety net on merge or in the merge queue. Avoid coverage-matrix selection; its data goes stale under churn. A test that never fails may run later or less often; that alone is no reason to delete it (see pruning in [auditing](auditing.md)).

## Fakes and test doubles

- The owner of a real implementation owns its fake, placed beside it at the dependency root: one shared fake per boundary, not one per caller.
- Run one contract suite against both real and fake, for example `describe.each([real, fake])` in TypeScript or a generic `fn contract_suite<T: Trait>()` in Rust, so fake drift fails visibly. Fakes throw on unsupported options.
- A lint rule flagging mocks of internal domain modules can enforce the skill's mock boundary; no existing rule was found, so this is a proposal.
- This complements, and does not widen, the skill's mock policy: DB and external-service doubles stay allowed; replacing time or scheduling still needs the complex-algorithm exception.

## Schema drift

When a project generates types, clients or fixtures from an API schema, or keeps HTTP fakes of its own API:

- Regenerate in CI and fail on any diff, so generated types cannot drift from the schema the server serves.
- Validate fake responses and fixture payloads against the same schema, so a fake cannot return a shape the server never sends.
- Add consumer-driven contract tools (Pact or similar) only once separately deployed consumers exist; a monorepo that builds both ends together gets the same signal from the two checks above.

## Snapshots

- CI never writes snapshots: Vitest refuses to write and fails on mismatched, missing or obsolete snapshots when `CI` is set; insta's `--unreferenced=reject` fails orphaned snapshots, and `cargo insta review` forces a decision per change.
- Prefer inline snapshots (`toMatchInlineSnapshot()`) for small values and cap the size of file snapshots; large ones go unread.
- A change that updates snapshots states why the new output is correct, not only that it changed.

## Ownership

Tests land in the same change as the code they cover. A team whose change breaks a product it did not test fixes the breakage and adds the missing test. Unowned flaky tests get ignored; route every quarantine or failure issue to a declared owner.
