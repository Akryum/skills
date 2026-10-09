# Scenario Patterns

Use patterns relevant to the current claim, language, and architecture. These illustrate stronger reasoning; they are not a mandatory test matrix. Reuse existing project infrastructure.

## A caught failure can pass

Weak: a forbidden operation throws, but the application catches it and the test checks only an empty result or settled loading. This can accept the forbidden operation happening.

Stronger: exercise the disabled/denied condition; observe zero relevant external requests or effects, no unexpected error, and usable state. Then enable/authorize the operation and verify expected result. Let the real dispatch/settlement boundary run before checking absence. Bypassing the guard must fail a relevant assertion.

Applies to disabled UI queries, canceled jobs, access control, and commands that must avoid destructive work in preview mode.

## An allowed external mock can hide wrong inputs

Weak: a database or remote-service mock returns the desired success regardless of resource identifier, write payload, filter, or permissions.

Stronger: keep the production operation real. Assert relevant external inputs and complete public outcome. An empty update, wrong target, or missing required field must fail. For multi-step workflows, use a shared stateful external fixture when needed: create/read, update one of two records/read both, delete/read again.

For filter/tenant isolation, inspect the actual combined predicate and bindings when using a DB mock; a truthy predicate cannot distinguish conjunction from disjunction. With an executing fixture or real DB, seed both allowed and denied matching rows and assert exact returned membership.

Mocked conflict/outage responses can test application error handling. Real database constraint or provider interoperability claims need corresponding integration proof. Do not require live infrastructure for every mocked-boundary test.

## Accepted or optimistic state is not completion

Weak: UI displays edited value immediately, API returns accepted status, or job gets queued; test ends before actual work can fail.

Stronger: await the appropriate completion signal, then verify result or stored effect through an independent consumer/read. For rejection, await failure/rollback, verify prior state and surfaced error, then prove a valid retry succeeds when recovery is claimed. A stateful external mock may provide simulated persistence; report that boundary accurately.

For jobs, distinguish submission from worker processing. For distributed workflows, distinguish a local send from recipient application of the result.

## Reuse exposes incomplete cleanup

Weak: resource disappears after delete or consumer closes; assertions never exercise the system afterward.

Stronger: recreate/reopen the same resource, mount/start a new consumer, or perform another operation. Verify the new state is visible and old subscriptions, handles, timers, or locks no longer interfere. Exercise actual runtime shutdown/disposal rather than calling the cleanup helper directly.

An unremoved deletion mask, stale lock, or retained subscription must fail this subsequent behavior.

## Synchronous observers can hide missing awaits

Weak: a callback records that related processing started, while the test checks neither pending state nor completed related output.

Stronger: delay a relevant external dependency through a permitted mock or controlled service. Start the real operation; verify it cannot settle successfully before required dependency completion. Release it and assert complete result. Removing the wait must fail early-settlement or missing-output assertion.

Applies to related data loading, enrichment, file uploads, batch completion, and workflow sequencing. Avoid replacing the local workflow itself.

## Delivery needs a recipient outcome

Weak: producer invokes a send callback; consumer/application handling is mocked or never exercised.

Stronger: execute real producer and consumer logic, with an allowed broker/service mock at the external boundary if appropriate. Publish a meaningful change and assert recipient-visible result. If ordering, undo, deduplication, or rejection is the contract, cover that observable effect too. Removing send or bypassing denial must fail recipient state or absence assertions.

For focus/presence, move from first target to second, then deliver stale first-target event. Assert active second target remains visible; counting a clear operation can accept the bug.

## Commands and jobs can succeed while doing nothing

Weak: command exits zero, job helper was called, or output file exists; actual contents/effects are unchecked.

Stronger: invoke shipped command/job entry with representative inputs. Assert meaningful output content, relevant exit status, and intended effects. Include unaffected input or destination to detect wrong-target changes. For failure/retry behavior, verify partial progress and duplicate effects according to the contract.

Examples: an export contains every expected record and required fields; a migration changes intended data only; a converter produces an artifact the intended reader can consume. Skipping work or writing empty output must fail.

## Generated artifacts need their consumer

Weak: generated source/config contains expected strings, while test calls the generator's dependency separately or never loads the output.

Stronger: produce actual artifact and consume it through the supported entry: load config, import/build generated code, install package, or parse exported file. Exercise meaningful configured behavior. Wrong registration, dropped options, or omitted required content must fail a consumer-facing assertion. Keep exact text checks when exact text or secret exposure is itself the contract.

## Partial outputs can hide data loss

Weak: parser checks only category/type; merged document checks a surviving substring; batch checks first result; copy checks initial equality; safe setter checks only absence of pollution.

Stronger: assert complete relevant fields and identifiers, exact preserved document contents, every distinct batch item, copy independence after mutation, and successful safe own-property writes. Check both the intended effect and forbidden side effect. Do not re-run the production algorithm to compute expected answers.

Partial matching is appropriate when omitted information cannot undermine the specific claim; use complete outputs for preservation contracts.

## Boundaries need values outside both sides

Weak: range includes every input; another range after every input is empty. Removing upper bound can still pass.

Stronger: inputs before, inside, and after interval; assert exact membership/order and documented inclusive/exclusive edges. For pagination, assert distinct page contents as well as totals. For quotas, include below/equal/above limit. For clocks, assert logical ordering as well as physical time when both matter.

Use supported deterministic input or injectable time sources. Fake scheduling still follows the skill's mocking rules.

## Type proof differs from runtime proof

Weak runtime claim: define helper, assert it exists, never call it.

Keep useful compile contracts in project's type/compile-test infrastructure, including rejected usages where relevant. For runtime support, invoke real public operation for meaningful variants and verify outputs/state. Evaluate a type-only claim with a compile regression, not an unrelated runtime defect.

## Incidental ordering can impersonate promised ordering

Weak: row identifiers, insertion order, and timestamps all imply the same sequence. Removing the production sort leaves the expected result unchanged.

Stronger: choose keys and insertion order that conflict with the required chronological/priority order. Assert complete records and the meaningful request sequence when replay order matters. Stable ordering of ties needs a check only if promised.

## Preservation requires something to preserve

Weak: opt-out starts with an absent cursor and asserts absence; expiry seeds only old records. Deleting the cursor or shortening retention produces the same result.

Stronger: seed existing metadata, disable its update, assert exact retention, and use the retained value through the next real consumer when relevant. Pair expired and still-valid records; verify removal and retention independently. Use supported time inputs or native short scheduling for basic lifecycle wiring; fake scheduling still requires the complex-algorithm exception.

## Approval and provider failure are distinct policy paths

Weak: one fresh client receives an explicit denial. This does not protect resource-specific checks after another successful action, or behavior when the provider throws.

Stronger: with the same supported session, perform an allowed action on one resource and a denied action on another; assert no denied effects and continued allowed use. Separately exercise thrown/unavailable provider results through the permitted service boundary. If the contract requires fail-closed behavior, assert no publication or mutation, then a legitimate successful retry. Do not assume every application treats provider failure as denial; preserve documented error semantics.

## Late completion must respect ended ownership

Weak: disposal occurs after all requests finish, or cleanup is checked only by a called helper.

Stronger: hold a real operation at an allowed external boundary, observe entry and pending public state, end its session/scope or leave its resource, then release the reply. Assert no revived membership, stale delivery, or mutation of a new visit. If re-entry is supported, demonstrate subsequent valid work. Keep old and new visits distinguishable; checking only a current identifier can accept stale completion after reacquisition.

Register failure cleanup immediately after the first acquisition, before another acquisition or fallible observation. Record original handles/PIDs independently; a faulty discovery helper must not hide resources from emergency teardown. Perform that fallback only after the scored observation, without masking release bugs.
