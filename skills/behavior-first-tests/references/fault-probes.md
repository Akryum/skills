# Targeted Fault Probes

Use when running isolated mutations or investigating false-green tests. Adapt to existing runner; these checks do not require a mutation framework or exhaustive campaign.

## Isolate writable tool state as well as source

- Capture pre-change tests and correct source for comparisons. Keep copies of source modules and public aliases together; verify the runner loads the copy rather than installed output or the original checkout.
- Shared dependencies can be read through links, but default paths inside `node_modules` can receive runner results, transformed configuration, prebundled modules, or build caches. Set supported cache/output/temp paths under the probe directory, disable unnecessary caches, or use a separate writable dependency view when the tool cannot redirect writes. Do not rewrite the shared dependency tree to accommodate a probe.
- Keep builds, generated modules, reports, and test fixtures in the isolated workspace. If an existing script writes elsewhere, adapt the isolated invocation before running it.
- Keep evidence of correct-source pass and relevant mutant failure, naming the source change and assertion. A helper that performs an extra await or repairs missing data can invalidate the claim even if source aliases are correct.
- Restore/remove only probe-owned source and artifacts. Preserve a useful failing regression artifact when a real production bug was discovered; report its path and proof boundary. Never leave an injected defect in the shared checkout.

## Choose a defect matching the claim

| Claim | Useful defect | What must reject it |
| --- | --- | --- |
| Input or response preservation | Mutate caller input or mock-returned object in place | Independently fixed expected fields, not an alias of the changed object |
| Correct outbound write | Drop a field or change target before the request | Request captured at the boundary plus relevant public outcome |
| Completed publication | Publish a Promise or partial results before an external reply | Public callback/state observed independently of helper Promise adoption |
| Resource release | Remove physical close/unsubscribe while retaining registry cleanup | Operation blocked by the leak, or forbidden stale delivery |
| Shared resource lifetime | Close on first consumer release instead of final release | Surviving acquired lease still blocks a competing exclusive operation; final release permits it |
| Complete recovery/reconciliation | Process only first eligible input | Two distinct eligible items both repaired, excluded items unchanged |
| Exact membership | Drop/add item, duplicate key, widen predicate | Complete membership with multiplicity/absence checks as required |
| Promised chronology | Remove sort | Keys/insertion order disagree with required timestamps; exact ordered contents/effects |
| Preserved metadata | Delete existing metadata on opt-out | Seeded nonempty value survives, and later consumer uses it |
| Retention | Shorten default lifetime | Expired record removed while newer record remains protected |
| Resource-specific policy | Reuse approval for another resource or ignore thrown failure | Allowed use succeeds; denied/error effects remain absent under documented semantics |
| Async ownership | Remove post-await ownership check | Held work completed after close/leave cannot revive or overwrite new ownership |

A defect that violates no promised observable behavior is equivalent for this contract. Do not force detection by asserting private implementation structure.

Verify reachability before treating a survivor as weak coverage. An application may protect its caller by cloning while an exported helper still corrupts its own argument or a later hook receives damaged transport data. Those are separate contracts: use a real supported consumer/extension or a focused exported-function test, without mocking the protective collaborator. Conversely, moving an internal abort notification can be harmless when the real dispatcher still awaits work and propagates errors; an observer seeing different timing does not itself establish a user defect.

## Probe both defects and harmless variation when justified

For strong request/data assertions, distinguish required ordering from incidental order. A filter conjunction or unordered key set may permit reordering; preserve all predicates/values and multiplicity rules without requiring original indexes. A small equivalent-order probe can expose overcoupling after tightening assertions. Do not mandate this probe for unrelated edits.

For actual bugs, first demonstrate the unmodified source failing the intended contract. If production fixes are outside the task scope, retain the isolated regression and report the bug instead of weakening the assertion or silently expanding scope.

When replacing weak tests, compare their actual inputs and observables before retiring them. For example, a new batch scenario does not cover a retired single-result normalization case. Exercise the distinct shape through the real public consumer, and use a targeted normalization fault if overlap is uncertain.

## Resource probes must leave bounded diagnostics

For a lifecycle probe, separate an explicit final-release assertion or native blocked-event error from a bounded contract poll and from the runner timeout. A timeout alone may show the suite rejects a leak, but is weaker evidence and can hide cleanup hangs. Keep acquired handles in failure-safe teardown; settle/cancel held external work without awaiting the broken completion indefinitely. Do not repair production release during the scored observation.

Record exact final source/test bytes or an equivalent immutable snapshot for scored probes. Re-run affected probes after meaningful changes: a previously effective deletion can become harmless when another path now supplies the same protection. Do not present an earlier mutation report as proof of different final code.

Teardown must cover setup failures after any acquisition and must not depend solely on the same discovery or termination helper being faulted. Preserve independently recorded handles and bounded fallback cleanup. Distinguish a named completion/rejection assertion from setup errors and whole-runner timeouts. For safe harness execution and fresh skill trials, use [execution and evaluation](execution-and-evaluation.md).
