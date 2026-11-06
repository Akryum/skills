---
name: behavior-first-tests
description: Write, improve, or audit tests in any software project for useful user scenarios, real behavior under test, appropriate database/external-service mocks, and demonstrated failure detection. Apply when adding regression coverage, reviewing test quality, auditing a large test suite, or verifying suspected bugs with fault probes across applications, APIs, libraries, command-line tools, and background jobs.
---

# Behavior First Tests

Write tests that protect something a user relies on. A green result must mean the claimed behavior works, and a plausible break in that behavior must make the test fail. Adapt to the project language, framework, architecture, and existing test runner; do not introduce a new stack merely to follow this skill.

## Required criteria

- Prioritize user scenarios. Users include people interacting with an application or CLI, developers consuming a library/API, and systems consuming a service or job output. A useful scenario does not require a UI.
- Test useful logic and outcomes. Name the capability or regression protected, and explain the consequence if it breaks.
- Keep application behavior under test real. Mocking databases and external services is allowed, including database drivers/ORMs, remote APIs, identity providers, object storage, payment services, and message brokers. Other mocking is allowed only when testing a complex algorithm; setup difficulty or async orchestration alone does not qualify.
- Assertions must reject broken behavior. Coverage percentages, passing baselines, snapshots, call occurrence, and test counts alone do not establish this.

## Choose the boundary before writing

Read the relevant production path and existing test harness. Identify which shipped code actually executes, which dependencies are substituted, and where failures become observable.

Choose the smallest boundary that executes the real behavior needed to protect the claim:

- Pure logic: call the real exported algorithm and assert complete outputs and important invariants.
- API or application workflow: execute the real public entry, business rules, and relevant local collaborators. Persistence or external-service dependencies may be mocked.
- Connector: execute the production client/adapter under test; its DB, SDK, HTTP, or external-service dependency may be mocked. Verify request targets, payloads, filters, results, and errors relevant to the claim. A mocked SDK does not prove SDK wire encoding; recorded SQL does not prove DB execution.
- UI/lifecycle: exercise the real framework lifecycle or runtime when focus, shutdown, disposal, or platform behavior matters. Manually calling a cleanup helper does not prove the runtime invokes it. For component behavior, mount real children and stores, act through user events, query by role or accessible name, and assert rendered output and emitted events. Reading or calling component internals (instance state, stub props, manually emitted child events) does not prove the user-facing contract.
- CLI or background job: invoke the actual command/job entry and inspect meaningful outputs, exit status, persisted effects, and retry behavior as appropriate. A called helper alone does not prove command wiring.
- Generated artifacts: consume the actual generated code, configuration, file, or package through its intended consumer. Text matching alone does not prove it loads or works.
- Types: use compile-time assertions and appropriate rejected usage cases. A function's existence is no runtime proof; keep compile-only claims separate.

A narrow pure test remains useful, but do not describe it as proof of a larger workflow it never executes. Add broader coverage where that workflow is the important contract.

Real entry points may clone data, normalize identities, or make a fallback harmless, shielding a distinct adapter/helper contract. Keep the useful workflow test and add a focused real exported-helper, adapter, or supported extension test only when that separate contract matters. Do not replace owned normalization or invent unsupported input merely to expose a fault.

## Preflight for each important claim

Before writing or replacing tests, record brief working notes covering:

- **Entry and boundary:** real consumer entry and owned logic executed; substituted dependencies and permitted mock boundary.
- **Contract:** user-visible outcome, relevant unchanged state, and distinct old contracts retained when replacing coverage.
- **Defect:** one plausible mistake that would violate that contract.
- **Inputs:** legitimate values or action sequence that make correct and faulty implementations differ.
- **Oracle:** decisive public assertion with independent expected values; identify which assertion should fail under the defect.
- **Lifetime:** when completion becomes observable and how acquired resources or held external work are released after failure, if applicable.

Use one compact note per important claim or related scenario group, not paperwork for every assertion. For an audit of existing tests, the finding format in [auditing](references/auditing.md) replaces these notes. Resolve unsupported inputs, hidden normalization, and an oracle already satisfied by initial state before coding. The distinguishing inputs below are part of this preflight; use only those relevant to the contract.

## Test doubles: allowed boundaries, useful assertions

Keep the client/adapter and local business rules needed for the scenario real. Supplying allowed DB/service doubles through globals, modules or generated configuration is permitted; the injection mechanism does not permit replacing owned behavior. Real infrastructure is not required merely to avoid these allowed mocks.

Other substitutions still need the complex-algorithm exception: fake internal application models/workflows, replacements for owned behavior under test, unrelated platform globals, or mocked module factories outside the allowed boundaries. An emulator remains a test double, not proof of actual browser behavior.

Allowed mocks must not conceal the defect the test claims to catch. A canned response is fine for response/error handling when paired with relevant request assertions. For target, payload, permission, or ordering correctness, inspect relevant inputs at the external boundary and assert public results; use a reusable stateful fixture when later reads or multi-step scenarios matter. Do not return an authored success row while ignoring all write inputs and call that persistence proof. Do not reproduce the production algorithm inside the mock to generate expected answers.

Prefer existing helpers over another fake. Real DB/service integration can add confidence when actual SQL execution or protocol interoperability is the contract, but is not mandatory for every scenario. Clearly distinguish validated application behavior with mocked dependencies from real DB execution or live-service compatibility.

For reusable stateful fakes, validate the dependency semantics scenarios rely on, such as filtering, writes, errors, and ordering. Where drift could invalidate several tests, use shared contract cases against the fake and real dependency when practical. Make unsupported operations fail clearly instead of silently returning success; do not build a duplicate service. Shared fakes, stubs, scripted transports and snapshot serializers are test infrastructure with tests of their own: cover the semantics consumers rely on, and make unexpected calls fail visibly and stay recorded even when production code swallows the error. Where an API schema exists, validate fake responses against it.

When provider interoperability matters, use an existing consumer/provider verification workflow, or supported schemas/sandbox checks for third-party services, where available. This checks interface compatibility; provider functional tests must establish persistence and business behavior. Do not mandate a new contract framework or live infrastructure for every mocked dependency.

Plain callbacks and event logs can observe real execution without replacing it. Prefer these when a mocking framework adds no value. A spy that calls the original implementation observes behavior; replacing its implementation or return value makes it a mock. Use observers only when the observed event/count is itself a public contract, and assert user outcomes too when needed.

For the complex-algorithm exception, state what algorithm is under test, why substituted inputs/dependencies are needed, and what remains real. Keep the algorithm real; assert complete outputs and invariants. Do not substitute its result or extend this exception to unrelated workflow tests.

Use deterministic real inputs and controlled clocks where the public API supports them. Replacing time/scheduling is a test double too; it requires the same algorithm exception. Prefer completion signals and bounded polling over arbitrary sleeps. For a blocked operation or held reply, distinguish its intermediate blocked state from expected final completion. Bound the final observation with a named contract assertion or documented external event, not only the runner timeout. Register handlers before triggering events. Protect resources from the first successful acquisition, including failures during later setup. Record handles independently of the discovery/cleanup behavior under test; settle or cancel held external work in teardown without waiting indefinitely on broken completion.

If external infrastructure is unavailable, use the allowed DB/service mocks to complete meaningful application scenarios. If a claim specifically requires real DB, service, or browser execution, report that missing proof separately. Do not mock owned application logic or weaken assertions to get green tests.

## Implement the preflight scenario

For an actual bug, write the failing regression first, confirm it fails for the intended reason, then fix the production code. For new behavior, use the same test-first approach when practical. Reuse shared setup/fixtures, but keep every value a test asserts on visible in its body: builders with overridable defaults, descriptive over DRY. One behavior per test, named for the behavior and its outcome; a name that needs "and" is two tests. Keep loops and conditionals out of test bodies; vary cases through a table of inputs and expected values.

Derive expected results from the product/API contract, domain rules, or independently known examples, and write them before running the code. Do not copy current implementation output or reuse the production algorithm as the oracle. Review snapshots against that contract before accepting updates; matching current output does not establish correctness. Prefer small or inline snapshots, and state why updated output is correct.

While implementing, treat existing tests as read-only. If one looks wrong against the contract, report it with evidence; do not edit, delete, skip or special-case its inputs to get green. Change it only within an in-scope contract change or bug fix, and say why the old assertion was wrong.

For data-content assertions, expected objects must not alias mutable caller inputs or mocked returns. Use independent literals or snapshots made before the action; capture mutable request payloads when received so later mutation cannot rewrite evidence. Caller input, transport data, and published metadata can be different objects: assert preservation where the named consumer receives it. A safe cloned caller does not prove shared payload remains intact for later callbacks or history.

Make inputs distinguish correct and broken implementations:

- Wrong-target risks need at least two distinct records; verify the intended target changes and the other survives. When an API separates target identity from patch identity, make them disagree through legitimate inputs or supported transforms; matching identities cannot prove which wins.
- Authorization/filtering needs allowed and denied records or clients; assert exact results and absence of denied effects.
  When relevant, distinguish explicit denial from thrown provider failures, and prior approval for one resource from access to another. Derive error behavior from the actual contract.
- Range/pagination needs values on both sides of boundaries, with exact membership/order rather than totals only.
- Ordering needs input, insertion, or storage-key order that differs from the promised output order. An already ordered fixture cannot reject a missing sort.
- Preservation or opt-out needs relevant nonempty existing state. An absent value cannot distinguish preserving it from deleting it.
- Retention/expiry needs both expired and retained records; an old-only example cannot reject an overly short default lifetime.
- Claims about processing all eligible items, including retry, refresh, reconciliation, or recovery, need at least two distinct eligible inputs. Assert every result/effect; include an unaffected or excluded item when selection matters. A single example cannot reject a first-item-only loop.
- Commands/jobs need meaningful output content and effects; a zero exit status or successful process launch alone can hide a no-op.
- Collaboration needs two consumers and visible convergence, including rejection, undo/redo, or stale events when relevant.
- Cleanup needs an observable consequence of actual release: no stale delivery or successful reuse that requires release. Reopening alone can succeed alongside leaked resources; choose an operation the leak would block. For shared-resource ownership, acquire it through two real consumers, release one and prove the surviving lease still holds it, then release the final consumer and prove release. Peer usability alone can hide premature close followed by transparent reacquisition.

For async ownership, hold an external reply, end the owning session/scope or resource visit, and reacquire when supported. Release the old reply and assert it cannot revive ended ownership or publish into the new visit; verify fresh work still succeeds. See [scenario patterns](references/scenario-patterns.md) for examples.

Assert after the relevant operation completes. Optimistic UI changes, accepted requests, or queued jobs do not prove completed success.

Inspect completion helpers too: an async helper returning a published Promise adopts it and can hide a missing await in production. For completion/publication claims, hold an external response and observe the actual public callback/state separately, then release it and check the final result. Read again through a fresh or uncached path when persistence is claimed. With a stateful external mock, this proves application behavior against simulated persistence; do not label it real DB persistence. For failure/recovery, check rejected effects stay absent, prior state remains usable, and subsequent valid work succeeds.

Use exact outputs for data-preservation claims. Partial matching is appropriate only when omitted fields cannot undermine the named contract. Check negative effects explicitly; a caught error, invisible row, or idle loading state may hide forbidden work. Preserve required ordering, but compare unordered sets or logically commutative filter operands without pinning incidental serialization order; reject extra, missing, or duplicate values as the contract requires.

Use property or stateful testing when varied inputs or action sequences add meaningful coverage, preferably with existing tools. Combine complementary invariants with concrete known examples; a single invariant or round-trip may accept matching bugs. For stateful comparisons, use a small independent contract model rather than duplicating production logic. Keep failing inputs/sequences reproducible and retain focused regressions; adding a framework is optional.

Choose supporting detail as needed:

- For diagnosing weak patterns, read [references/scenario-patterns.md](references/scenario-patterns.md).
- For concrete generic test examples and the faults each should catch, read [references/examples.md](references/examples.md). Adapt the illustrative APIs to the project; do not copy a new harness merely to match an example.
- For research evidence, tradeoffs, and limits of these recommendations, read [references/research.md](references/research.md).
- For browser/process harnesses or independent evaluation of this skill, read [references/execution-and-evaluation.md](references/execution-and-evaluation.md). Keep execution constraints portable and trials uncontaminated.
- For any fault probe, read [references/fault-probes.md](references/fault-probes.md).
- For audits of existing suites, read [references/auditing.md](references/auditing.md); for consequences inside a running service, [references/live-verification.md](references/live-verification.md).
- For suite infrastructure (mutation testing in CI, arid skip lists, flaky quarantine, fake ownership, snapshot policy), read [references/suite-practices.md](references/suite-practices.md).

## Prove failure sensitivity

For each important claim, identify a realistic defect that should fail the test. For high-risk boundaries, weak assertions, or tests proposed as regression proof, run a targeted fault probe when practical. Do not require an exhaustive mutation campaign for routine changes.

1. Run the focused baseline and confirm the intended source implementation is loaded. Rerun new tests several times; one that flips on unchanged source is not accepted until its waits or state are fixed. A test flaking near 1% of runs no longer works as a signal; investigate an existing flaky test before quarantining it, since a new flake can expose a production race.
2. Probe a plausible production defect in isolated source/aliases with private writable caches, generated modules, temporary paths and reports. Dependency links alone do not isolate writes. Never inject defects into a shared checkout or dependency tree. Keep probes in a scratch folder run through the project's own runner configuration.
3. Require an assertion failure tied to the claimed behavior. Syntax/import failures, unrelated timeouts, and setup failures are not proof.
4. For a survivor, verify reachability and demonstrate a promised observable defect with legitimate input; do not score equivalent changes. Strengthen genuinely weak scenarios, then confirm correct-source pass and defect failure. Never kill a survivor with a change-detector test (logging, timeouts/backoff, flags, capacity constants); record why it is not worth a test. Protective normalization and distinct adapter contracts are covered in [fault probes](references/fault-probes.md).
5. Restore/remove only probe-owned artifacts; preserve concurrent work. Keep scored source/test snapshots current after meaningful changes.

Read [references/fault-probes.md](references/fault-probes.md) before writing any probe. It holds the probe contract (assert the correct contract, passing control in the same file, only assertion failures count), the out-of-checkout harness recipe, the defect-per-claim table, and helper/oracle isolation.

If a probe cannot run, state the proposed defect and that sensitivity remains unverified. Do not claim a static review or green baseline proves it.

## Audit an existing suite

For an audit request, report concrete gaps with file/line, escaping defect, stronger user scenario, and whether the finding was demonstrated or inferred. Audit does not authorize rewriting tests. Delegate only when requested or otherwise authorized. Follow [references/auditing.md](references/auditing.md) for slicing, finding format, evidence per claim type, status and severity scales, pruning, and report shape.

- Audit in a context separate from the one that wrote the tests. Report only findings that affect fault detection; skip style nits. A "cannot fail" or "redundant" claim is demonstrated only by a surviving mutant or a replaced function body that stays green.
- Look first for tests that assert current buggy behavior as intended, and for shared helpers, fakes or fixtures that ignore inputs (filters, sort, pagination, HTTP method, body) or swallow unexpected calls. One such helper hides defects in every consumer.
- Group a repeated weakness once with all occurrences. Deduplicate production bugs by root cause and give each one owner location; cross-reference related causes.
- Verify every production-bug claim with a bug probe before reporting it as fact; record status (confirmed, partial, latent, duplicate, refuted, unverified) and severity (critical, high, medium, low). Refute or downgrade claims the probe disproves; keep the correction visible.
- Verify server-side consequences against a live service only on a development instance the user named, with scratch resources, sentinel data and a cleanup log; lockout and irreversible tests need explicit approval. See [references/live-verification.md](references/live-verification.md).

## Finish within scope

For a write/fix request, review the final tests and remove duplicates, vacuous assertions, and implementation-only coupling. Reject tests that can pass without executing the claimed behavior, swallow unexpected failures, or accept both correct and faulty outcomes, and tests with no assertion, trivial getters or pass-throughs, or unrelated behaviors combined in one test. Each added test must protect a useful contract or distinct fault case beyond existing coverage. When replacing or combining existing scenarios, first map each distinct original assertion to its retained check. Preserve public input shapes, sparse/partial service replies, omitted fields, failure cases, and completion states; similar names or outcomes do not establish duplicate coverage. Preserve meaningful combinations too: collection+nested coverage and singular+flat coverage do not establish singular+nested coverage. If a broader workflow hides an adapter contract because fallback has no visible effect, retain the focused real adapter assertion rather than replacing it with an indistinguishable outcome. Mark a retired assertion redundant only when another check detects the same plausible fault; do not assume fuller fixtures subsume sparse or partial replies. If a test fails, diagnose the contract and production path before changing expectations; never weaken assertions or bless current buggy output merely to make it pass. Run focused checks and applicable required gates. Broaden validation only when risk or observed failures justify it.

Keep the final report compact: protected behavior, each new test with the fault it catches, validation performed, DB/service or allowed algorithm mocks and their proof boundary, fault-probe result, and remaining infrastructure gaps. Distinguish algorithm/API/type proof from platform integration, actual persistence, external interoperability, or multiple-consumer proof. Do not stage, commit, or push without authorization.
