# Auditing a Test Suite

Use when reviewing many test files at once or coordinating several reviewers over one suite. Per-claim preflight notes do not scale to hundreds of files; the fixed finding format below replaces them. An audit is read-only: it reports, probes in scratch space and proposes regression tests. It does not rewrite tests or fix production code unless the user separately authorizes that.

## Workflow

1. **Inventory and slice.** List test files with the production code they target and the shared helpers, fakes and fixtures they import. Slice by owner module or layer, keeping each test with its production code. Size slices so a reviewer reads every test line and the targeted production code; broad slices get skimmed, large files first. A broad first pass followed by smaller, deeper slices works when budget is uncertain.
2. **Give shared code one owner.** Each shared helper, fake, fixture factory, serializer or setup file belongs to exactly one slice. Other slices cite it with their consumer count instead of re-auditing it.
3. **Share one brief.** Every reviewer gets the same reading list (this skill, project testing standards, harness docs), read-only constraints, scratch probe folder, test-run budget, finding format, output path and a word limit for the returned summary. A later wave verifies the earlier findings for its slice instead of repeating them. Audit tests from a context separate from the one that wrote them: give the reviewer code and contracts, not the author's reasoning. Delegate only when requested or otherwise authorized.
4. **Review each slice.** Read every test line by line along with the code it targets. Check the high-leverage targets below first. Hunt production bugs in branches tests never reach: operation-key typos, missing awaits, wrong-target lookups, off-by-one, null/empty handling, coercions, dropped filters, stale async publication. Run the [bug-hunt sweep](#bug-hunt-sweep) on every function it applies to; it is required, not a suggestion. List coverage gaps: user-relied logic with no test that can fail for its main defect. Take every cited line number from a fresh numbered read of the current file.
5. **Consolidate.** Deduplicate by root cause, not symptom or file: one id, the owning location, other occurrences listed under it. Cross-reference distinct root causes usually fixed together. Group a repeated weakness once with every `file:line` occurrence. Assign every cross-slice lead (a bug whose caller or consequence lives in another slice) to exactly one verifier, and have the other slice reference that entry instead of rating it again.
6. **Verify before reporting a bug as fact.** Every production-bug claim gets a bug probe ([fault probes](fault-probes.md)) from a verifier who re-reads the code at the current revision and treats earlier conclusions as leads. Trace reachability from a user entry. Assign a status and re-rate severity from what was observed. A critical or high rating that depends on server, database or other-service behavior read from source stays provisional: confirm it with a live request ([live verification](live-verification.md)) or record it as partial with the server step marked inferred. Reading server code has both missed rejections that made a bug harmless and missed consequences that made one worse.
7. **Report.** Produce a bug register and a test-rework list (below). Keep inferred findings visibly separate from demonstrated ones.

Verification changes results. In one audit of roughly 950 test files, about a third of the reported production bugs ended latent, duplicate or refuted after probing, and live checks re-rated several survivors. Unverified audit claims should not drive fixes. An independent rerun of the same suite with this workflow was far more precise (verification moved only 7% of review claims) but found about half as many bugs. Every critical and high miss sat in a file the reviewer had read; the misses clustered on input classes and concurrency scenarios nobody tried, and on guards filed as "untested" instead of probed. The sweep below targets exactly those.

## High-leverage targets

One finding here often explains many escaped bugs.

- **Tests that pin buggy behavior.** A test that asserts, as intended, what the contract calls a defect: "skips null values" in a query or filter serializer, a guard flag that never resets, a type test locking a wrong return type. For every verified bug, search for tests that pin it; they change in the same fix.
- **Shared fakes that ignore inputs.** Stubs, in-memory stores and scripted transports that answer the same regardless of filter, sort, pagination, single-versus-many key, HTTP method, body or query; key responses by path only; fail every method when one should fail; throw on unexpected requests that production code swallows, without recording them; form or validation stubs that never run the schema; snapshot serializers that drop `undefined`, collections or NaN. Count consumers. Fix the helper first, then triage the new failures; never weaken them.
- **Fixtures that cannot distinguish.** Fields production never sets, already-sorted rows, identifiers equal to names, single records, empty initial state, one-sided bounds.
- **Tests that exercise no production code.** A copy of the algorithm inside the test, assertions on source text, mocks of owned stores or services that replace the behavior under test.
- **Tests that can never fail.** Pre-satisfied assertions, a mocked translator or formatter returning the key under test, throttling configured as passthrough, negative type tests that compile either way, cleanup checks that skip the relevant keys.

## Bug-hunt sweep

Reading code finds the defects a reviewer thinks of; the sweep finds the ones they don't. For each function that matches a row, try every listed input against the contract, at least by reading and by a probe when the outcome is unclear. Record in the slice report which functions were swept.

| Function kind | Inputs to try |
|---|---|
| Serializers, query/filter builders, encoders | `null`, `undefined`, absent key, `''`, `0`, `false`, `NaN`, `[]`, `{}`, `[x, undefined]`, nested empty `_and`/`_or`, Date/Temporal/Map/Set, very large numbers |
| Parsers, route/query readers | special and reserved keys (`__proto__`, `constructor`), duplicate keys, numeric-looking strings, unexpected types, malformed values |
| Path, URL and key builders | `/`, `?`, `#`, `%`, `.`, `..`, spaces, unicode, empty string, composite or array keys |
| Editors of structured values (filter rows, rules, forms) | objects with several keys or operators, existing values the editor does not create itself, clearing one entry of a list, removing the last entry, switching type or operator with a value present |
| Text and file inputs | non-ASCII and emoji, very large payloads, pasted or programmatic input that fires no key events |
| Numeric inputs | bounds, negative, decimals on integer fields, precision and scale limits, exponent forms |

**Concurrency and lifecycle scenarios** for every stateful composable, store or component that saves, fetches or navigates:

- an edit or second request while a save or fetch is in flight;
- a reply that lands after its inputs changed (prop, selection, route, workspace) or after unmount;
- a failure partway through a multi-step write, then a retry;
- an error raised after navigation or a redirect;
- a second subscriber or instance while the first unmounts.

**Guards are bugs until probed.** When a test finding concerns an untested lockout, last-method, last-admin, permission or validation guard, probe the guard itself against the other actor's or the server's state before filing it as test-only. "The guard is untested" and "the guard is wrong" need different evidence, and the worst defects hide in that gap.

**Comments are not contracts.** A comment that documents a silent failure, a "best-effort" step or a behavior inconsistent with the rest of the feature does not refute a bug. Rate the user consequence. Record the comment as context, and mark the entry "needs product decision" only when the behavior is plausibly intended and harmless to the user.

## Finding format

- Test finding: `file:line` | problem | escaping defect | stronger user scenario | demonstrated or inferred.
- Production bug: `file:line (function)` | trigger input | user consequence | reachability | demonstrated or inferred.
- Coverage gap: `file:function` | missing user scenario | risk.

Report only findings that affect fault detection or trust in the signal. Skip style nits, naming preferences and tests for cases no caller can reach; a reviewer asked for problems always finds some.

Evidence by claim type:

- **Semantic** ("cannot fail", "redundant", "weak oracle"): demonstrated only by a surviving mutant or a pseudo-tested body (replace the covered function body with a default return and the suite stays green). Without that, report it as inferred and name the probe that would settle it.
- **Structural** (no assertion, assertions only on mock calls, oversized or unreviewed snapshots, logic computing expected values inside the test, setup hiding the values asserted, duplicated fixtures or fakes): may rest on reading the code; still cite `file:line`.
- **Missing test for a survivor:** propose one only if it is not a change-detector ([fault probes](fault-probes.md)).

Verified register entry: status, severity, location, reachability (caller chain from a user entry with `file:line`), reproduction, expected, actual, evidence (probe path, command, decisive output line verbatim), root cause, suggested fix (not implemented), regression test (where it belongs and its user scenario), tests hiding it, source report. List refuted and duplicate entries last.

## Status

- **confirmed:** the probe fails on the defect through a user-reachable path.
- **partial:** defect demonstrated at function or component level; one step to the user (UI wiring, server response) is inferred and named.
- **latent:** defect demonstrated, but no current caller reaches it; say why.
- **duplicate:** same root cause as another id; name it.
- **refuted:** the claimed behavior does not occur; cite the passing probe or the code preventing it.
- **unverified:** the probe could not run; state the exact blocker after a real attempt. Use sparingly.

## Severity

Rate the consequence on the reachable path, not the code smell. A latent bug keeps the severity it would have when reached but is listed separately.

- **critical:** persistent data loss or corruption, permission bypass or broadening, account or admin lockout, a destructive operation applied wider than requested.
- **high:** wrong data saved, duplicate writes, silent failure the user believes succeeded, security-adjacent defects (prototype pollution, credential mix-up).
- **medium:** broken feature or wrong display with a workaround.
- **low:** cosmetic, unusual edge input, performance.

Test findings: **high** when a test pins a defect, a shared helper hides defects for many consumers, or a test of a critical contract exercises no production code; **medium** when a test cannot fail for its main defect; **low** for duplication, coupling or redundant weak checks.

## Refute and downgrade

Refute or downgrade when the probe passes, a guard elsewhere blocks the path, validation rejects the input first, the server rejects the request, or the consequence is narrower than claimed. Upgrade when a weak test turns out to hide a real bug. Record each correction beside the original claim so readers of earlier reports see it; keep refuted entries so they are not reported again.

## Pruning

Most deletions in practice are obsolete tests whose subject was removed; tests judged redundant often still catch faults nothing else does.

- **Obsolete:** delete in the same change that removes the subject.
- **Redundant:** propose deletion or merge only when the test catches no fault another test misses, shown by a probe or mutation run. Similar names, overlapping coverage or "never fails" are not evidence.
- **Pins of known-wrong behavior:** keep, but label them in name or comment with the issue (for example `currently_*` or `BUG(#123)`), and flip them in the fix. Unlabeled pins of incidental output (ordering, formatting, generated ids, internal call sequences) are rewrite or deletion candidates.
- **Change-detectors** that break on refactors without catching user-visible bugs, and tests flaky past their quarantine limit ([suite practices](suite-practices.md)), are deletion candidates once their contract, if any, is covered elsewhere.

## Report shape

- **Bug register:** totals by status and severity, critical items one line each, corrections to earlier reports, related root causes, limits (what stayed inferred or untestable).
- **Test rework by category:** tests pinning a verified bug (fix with the bug); tests that miss a verified bug in their area (rework inputs); tests exercising no production code (delete or rewrite); shared helpers hiding defects (fix first); duplicates (merge only with the pruning evidence above). Label estimated counts as estimates.
- **Coverage gaps** ranked by risk.
- **Audit accuracy:** the share of reported bugs that ended refuted, duplicate or latent after verification, and the result of spot-checking a sample of structural findings (LLM smell detection is right only about three times in four; see [research](research.md) section 15). Turn repeated false-positive patterns into skip rules or checks kept with the project, so the next audit does not report them again.
- **Suggested order:** regression tests and fixes for critical bugs, then shared helpers, then pinned tests alongside their bugs, then remaining bugs by owner, then gaps.

Keep probes, registers and logs outside the repo unless the user asks to land regression tests.
