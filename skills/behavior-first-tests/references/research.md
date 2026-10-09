# Research: Tests That Protect User Behavior

Initial source-based research synthesis, 2026-10-07. Scope: user-oriented scenarios, mock boundaries, assertions/oracles, fault sensitivity, property/stateful testing, and agent-written tests across software projects. Sources are research papers, their authors' publications, and official practitioner/tool documentation. This is a focused synthesis, not an exhaustive systematic review or a launched background research job.

The recommendations below distinguish empirical findings, practitioner guidance, and our proposed agent workflow. The skill's allowed-mocking policy comes from user requirements; it is not claimed to be a universal conclusion of research.

## 1. Start from observable contracts, not internal method coverage

Testing Library's principles emphasize tests resembling intended use and observing rendered DOM instead of component internals. This is frontend guidance, not evidence that every project needs browser tests. [Testing Library guiding principles](https://testing-library.com/docs/guiding-principles/).

Google's public-API guidance argues that behavior-preserving implementation changes should usually leave assertions intact. It also acknowledges cases where interaction details matter, such as whether a cache avoids a datastore read. [Test Behavior, Not Implementation](https://testing.googleblog.com/2013/08/testing-on-toilet-test-behavior-not.html).

Playwright independently recommends user-visible behavior, isolated test state, user-facing locators, and retrying assertions. It explicitly permits network mocking for third-party dependencies. Auto-waiting for UI assertions helps settlement but does not itself prove server persistence. The latter is our boundary inference. [Playwright best practices](https://playwright.dev/docs/best-practices).

**Recommendation:** identify consumer action, observable result, relevant absent side effects, and a plausible escaping defect. For libraries, services, and CLIs, consumers use their public API, protocol, or command rather than a DOM. Prefer the cheapest boundary that actually executes the claimed behavior.

## 2. Mocking is a boundary choice; fidelity determines what it proves

Google's test-double chapter distinguishes real implementations, fakes, stubs, and interaction-oriented mocks. It treats fidelity as resemblance to the relevant dependency contract, recommends testing reusable fakes, and describes running shared contract tests against fake and real implementations. It does not require perfect simulation of every dependency feature; unsupported paths can fail fast. [Software Engineering at Google, chapter 13](https://abseil.io/resources/swe-book/html/ch13.html).

Fowler separates state verification from interaction verification. Interaction checks can be useful where state cannot establish the contract, such as avoiding an unnecessary cache fetch; mock-driven verification also introduces coupling tradeoffs. [Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html).

**Recommendation:** DB/external-service mocks are allowed. Keep the owned workflow or adapter under test real. A response stub is appropriate for response/error handling; target/payload/filter correctness also needs relevant outbound assertions. A stateful fake helps multi-step application scenarios. Do not infer real SQL execution, durability, or provider interoperability from it. Test shared fake fidelity where divergence would invalidate many consumers; do not build a full duplicate service.

## 3. Consumer/provider contract checks close some mock blind spots

Pact describes consumer tests sending real requests to a mock provider, which matches expected requests and supplies responses. Provider verification replays the recorded interactions against the provider implementation. This checks compatibility without running all services together. [How Pact works](https://docs.pact.io/getting_started/how_pact_works).

Pact explicitly separates interface compatibility from provider business behavior: matching request/response contracts does not prove an order was actually persisted correctly. The provider's functional tests own that claim. [Contract tests vs functional tests](https://docs.pact.io/consumer/contract_tests_not_functional_tests).

**Recommendation:** use contract verification when a project controls both ends or has an established provider-contract workflow. For third-party providers, use their supported schemas/sandbox checks where relevant and available. Keep this conditional; a small app does not need a contract framework for every dependency. Never label a mocked success response as provider persistence proof.

## 4. Coverage exposes gaps but is insufficient as a quality target

Inozemtseva and Holmes studied 31,000 suites across five large Java systems. With test count controlled, they found low-to-moderate correlation between coverage and mutation-based effectiveness. Their conclusion supports coverage for identifying under-tested code, with caution against treating it as a quality target. The study's effectiveness measure was mutation testing, not an unlimited sample of real production bugs. [Coverage Is Not Strongly Correlated with Test Suite Effectiveness, ICSE 2014](https://cs.uwaterloo.ca/~rtholmes/papers/icse_2014_inozemtseva.pdf).

**Recommendation:** use coverage to locate unexecuted branches, then ask what assertions distinguish correct behavior from plausible defects. A new user scenario can matter without increasing line coverage. Avoid quota-driven tests whose only contribution is executing a line or asserting an object exists.

## 5. Mutation testing adds evidence of defect detection, with limits

Just and colleagues evaluated 357 real faults across five applications, using developer-written and generated suites. They found statistically significant association between mutant detection and real-fault detection independent of coverage. This supports mutation analysis as a useful adequacy signal, not a guarantee that every mutant represents a realistic fault or that every passing test is useful. [Are Mutants a Valid Substitute for Real Faults in Software Testing?, FSE 2014](https://homes.cs.washington.edu/~rjust/publ/mutants_real_faults_fse_2014.pdf).

A Google industrial approach scales mutation testing by applying it incrementally to changed code, filtering less useful mutations, and limiting feedback. This supports bounded, actionable checks rather than mutating an entire repository for every change. [Practical Mutation Testing at Scale, 2021](https://research.google/pubs/practical-mutation-testing-at-scale-a-view-from-google/).

A separate analysis of 15 million mutants reports developers improving suites over time and evidence linking mutants with historical real faults. This is industrial observational evidence; it does not establish a universal causal improvement rate for every team or workflow. [Long Term Effects of Mutation Testing, ICSE 2021](https://research.google/pubs/long-term-effects-of-mutation-testing/).

**Recommendation:** first reproduce an actual bug when available. Otherwise choose a small number of plausible changed-code defects: wrong target, missing field, bypassed guard, incomplete result, missing wait/cleanup, or wrong boundary. Require a relevant assertion failure against the faulty implementation and a pass against correct code. Track what was actually probed.

## 6. A survivor is not automatically a missing test

Stryker documents equivalent mutants: some syntactic changes do not change observable behavior, preventing a meaningful test from distinguishing them. A forced 100% mutation target can therefore be unproductive. [Equivalent mutants](https://stryker-mutator.io/docs/mutation-testing-elements/equivalent-mutants/).

**Recommendation:** investigate a survivor in this order: did the mutated source load; did the affected behavior execute; did the mutation change the required observable contract; were assertions strong enough? Only then add a scenario. Build/import failures and unrelated test failures do not establish behavior sensitivity. Mutation tools may classify loop-induced timeouts as detection; a timeout deliberately tied to a promised termination property can be useful, but an incidental timeout is not the claimed assertion proof.

## 7. The oracle matters as much as input variety

The oracle survey defines the central problem as determining whether observed behavior is correct. It reviews specifications, models, contracts, metamorphic relations, and human/domain knowledge as possible sources of expected behavior. [The Oracle Problem in Software Testing, 2015](https://discovery.ucl.ac.uk/id/eprint/1471263/).

**Recommendation:** derive expectations from product/API contracts and domain examples rather than copying current implementation. For a sorting API, sortedness alone accepts empty output; preservation alone accepts unchanged input. Combine relevant invariants with known examples. A round-trip alone can miss paired encode/decode mistakes; add known external representations if format compatibility matters. These examples are our deductions from oracle requirements, not measured results from the survey.

## 8. Stateful/property testing can explore sequences beyond named examples

Hypothesis stateful testing generates action sequences as well as input values. Its documentation demonstrates comparing operations on an implementation with a simpler independent model, and notes simpler generated-data tests may suffice for less complex cases. [Hypothesis stateful tests](https://hypothesis.readthedocs.io/en/latest/stateful.html).

**Recommendation:** use existing property-testing tools for transition-heavy contracts such as create/update/delete, subscribe/dispose, retry/duplicate delivery, or undo/redo. Maintain a small independently specified model; do not replicate the implementation wholesale. Keep focused known regressions alongside generated exploration, and retain reproducible counterexamples. This is conditional guidance, not a mandate to add a framework.

## 9. Labels such as unit/integration do not reliably describe proof boundaries

A 2025 industrial classification study motivates more precise characteristics because conventional test categories vary in interpretation and can obscure what a test executes. Its scope is classification of tests, not proof that one test level universally detects more defects. [What Types of Automated Tests do Developers Write?, AST 2025](https://homes.cs.washington.edu/~rjust/publ/test_type_analysis_ast_2025.pdf).

**Recommendation:** describe entry point, shipped code exercised, substituted dependencies, and observable outcome. A test named integration may still mock every important collaborator. Conversely, a small library test can protect a complete user contract. Avoid fixed test-pyramid ratios unsupported by the project's risks and costs.

## 10. Agent-written tests need intent and execution gates

Uber's 2026 AutoCover report describes repository-aware preparation, scenario-guided generation, execution, validation, and repair. Its validator uses bounded mutations to guide better inputs/oracles and accepts useful scenario coverage alongside code coverage. The paper notes build/generated-artifact context and reliability concerns, and acknowledges limitations of coverage proxies and subjective expert assessment. This supports an agent workflow informed by actual project artifacts; it does not establish that a generic prompt or this skill alone guarantees quality. [Automated Software Test Generation at Industry Scale, ICSE-SEIP 2026](https://homes.cs.washington.edu/~rjust/publ/auto_cover_icse_2026.pdf).

**Recommendation:** agents should inspect actual entry points and harnesses, state scenario intent before writing, execute focused checks, probe relevant faults when warranted, and avoid repairing failing tests by merely changing expectations to current output. This is our workflow synthesis, not a reproduced AutoCover evaluation.

## Proposed review questions

These questions operationalize the evidence; they are not an established numerical scoring system:

1. Which user/API/command capability would break if this assertion were absent?
2. Does the test execute the shipped entry needed to prove that capability?
3. Which dependencies are mocked, and what compatibility/storage claims remain unproved?
4. Do inputs distinguish wrong targets, omitted values, and relevant boundary/denial cases?
5. Is expected output independently grounded in a contract or domain example?
6. Does assertion observe completion and required side effects, not merely initiation?
7. What plausible defect was demonstrated to fail? Was source loading checked?
8. Could harmless refactoring break assertions without changing required behavior?
9. Is a new test adding a useful contract, or duplicating an existing scenario?

## Remaining questions for deeper empirical work

- Optimal mix of targeted manual probes and automated mutation tools for small agent changes.
- How well mutation results generalize to authorization, async/lifecycle, and distributed defects across stacks.
- Cost and effectiveness of maintaining fake/provider contract parity in smaller projects.
- How to measure scenario contribution without incentivizing duplicate tests or artificial count targets.
- How independently generated expectations affect agent-test oracle quality.

No universal answer is established here. Avoid turning these open questions into hard thresholds or claiming causal effectiveness beyond the cited evidence.
