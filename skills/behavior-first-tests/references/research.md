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

Sections 11–18 extend this synthesis (2026-11-06) with evidence on test quality at scale: industrial mutation testing, coverage, generated tests, agent oracles, LLM audits, deletion, flakiness, doubles and snapshots. Industrial papers report the authors' own deployments; blog posts and single preprints are marked as weaker evidence where cited.

## 11. Diff-scoped mutants changed developer behavior; suppression made them usable

Google's pipeline mutates only lines changed in a changelist that some test covers, skips "arid" AST nodes, generates at most one mutant per line, and posts survivors as code-review findings with "Please fix" and "Not useful" feedback. Across 776,740 changelists in 10 languages, this cut the median from 820 candidate mutants per changelist to 7, with a median of 2 surfaced. [Petrović et al., TSE](https://arxiv.org/pdf/2102.11378v1). One mutant per line is justified empirically: in more than 90% of cases every mutant on a line is killed or none is. [Petrović et al., ICSE 2021](https://arxiv.org/pdf/2103.07189).

Developers first rated 85% of surfaced mutants unproductive: some were equivalent, others killable only by a brittle change-detector test, such as asserting a collection's initial capacity. More than 100 arid rules followed. The first three, covering logging, time-related code (deadlines, timeouts, backoff) and configuration flags, raised the productive share from about 15% to 80%; later rules reached 89%. The authors call suppression critical to adoption. [TSE](https://arxiv.org/pdf/2102.11378v1). Learned suppression (MuRS) produced a modest gain in an A/B test, 11.45% versus 12.41% negative feedback, while also dropping about 12% of useful mutants; treat it as a small effect. [MuRS, FSE 2023](https://arxiv.org/html/2306.09130v1).

Over six years and about 15M mutants, developers exposed to more mutants wrote more and stronger tests (fewer survivors on later changes, new tests killing more than the targeted mutant), while changed-code coverage did not rise. About 70% of high-priority bugs sat in already-covered code and coupled to live mutants, mostly from statement-block removal, the bluntest operator. This is observational evidence from one company. [ICSE 2021](https://arxiv.org/pdf/2103.07189). Google never gated merges on a score; absolute mutation scores are distorted by equivalent mutants and described as "not so useful" targets. [PIT report](https://arxiv.org/pdf/1901.03865).

**Recommendation:** probe changed, covered behavior and report specific survivors, never a global score. When no claim-specific defect is obvious, start with statement-block removal on the changed code. Before writing a test to kill a survivor, ask whether it would be a change-detector; logging, timeouts/backoff, flags and capacity constants usually are. See [fault probes](fault-probes.md) and [suite practices](suite-practices.md).

## 12. Coverage gates discard useful tests; assertions and replaced bodies are sharper signals

Google computes coverage for about a billion lines a day, and 45% of surveyed developers use it often while authoring. [Code Coverage at Google](https://research.google/pubs/code-coverage-at-google/). Its testing book still calls coverage weak: it shows a line ran, not what was checked, and an 80% target gets treated as a ceiling. [SWE at Google ch. 11](https://abseil.io/resources/swe-book/html/ch11.html). Assertion count and assertion coverage correlate more strongly with effectiveness than coverage. [Zhang & Mesbah, FSE 2015](https://people.ece.ubc.ca/amesbah/resources/papers/fse15.pdf). A later study cautions that correlations computed on randomly sampled pseudo suites are inflated, which weakens all such correlation figures, including section 4's. [Zhang et al., UCL](https://discovery-pp.ucl.ac.uk/10075196/7/Zhang_Do%20Pseudo%20Test%20Suites%20Lead%20to%20Inflated%20Correlation%20in%20Measuring%20Test%20Effectiveness_AAM.pdf).

Meta's ACH found that 49% of its mutant-killing tests added no line coverage; a coverage gate would have discarded half the useful tests. [ACH](https://arxiv.org/html/2501.12862v1). Pseudo-tested methods, covered methods whose whole body can be replaced without any test failing, appeared in all 21 well-tested Java projects studied; required methods averaged a 52% higher mutation score. [Vera-Pérez et al.](https://arxiv.org/pdf/1807.05030). Classic smell detectors are unreliable (one tool misclassified more than 70% of smells) and miss tests that assert nothing or setups that still throw. [Panichella et al., EMSE 2022](https://pure.tudelft.nl/ws/portalfiles/portal/137994226/s10664_022_10207_5.pdf).

**Recommendation:** to show that a test cannot fail, replace the covered function body with a default return or produce a surviving mutant; a smell-tool flag or a coverage number is not that evidence. Accept a scenario that kills a new fault even when it adds no coverage.

## 13. Generated tests became useful only behind fail-then-pass filters

Meta's TestGen-LLM discards a test that does not build, does not pass on first run, fails any of five repeated runs, or adds no line coverage. On Instagram, 75% of generated tests built, 57% passed reliably and 25% increased coverage. Engineers accepted 73% of recommendations they received, yet only about 10% of classes improved (the abstract's 11.5% conflicts with the body). Reviewers rejected a test with no assertion, a trivial getter test, and two tests that violated single responsibility. [TestGen-LLM](https://arxiv.org/html/2402.09171v1).

ACH replaces the coverage gate: a test must fail on a concern-specific mutant, pass on the original, and kill a mutant no existing test kills. An LLM judge discards equivalent mutants; its precision and recall rose from 0.79/0.47 to 0.95/0.96 once misleading code comments were stripped. Engineers accepted 73% of tests and asked to see the concrete fault each test catches. [ACH](https://arxiv.org/html/2501.12862v1). SWT-Bench uses the same fail-to-pass criterion for issue-derived tests. [SWT-Bench](https://alphaxiv.org/abs/2406.12952).

Both Meta systems use current behavior as the oracle, so they produce regression guards and cannot find existing bugs. **Tension:** the skill keeps its rule that expected values come from the contract, not current output. A current-behavior oracle is acceptable only as an explicitly labeled pin (section 16), never as proof of correctness.

**Recommendation:** accept a new test only after it fails on a relevant defect, passes on correct source, and passes on reruns. Present each test with the fault it catches. Before presenting, reject tests with no assertion, trivial getter or pass-through tests, and tests mixing unrelated behaviors. Judge mutant equivalence from code and contract, not comments.

## 14. Agent oracles copy actual behavior, and agents tamper with tests

LLM test generation tends to write oracles that capture actual rather than expected behavior, and models struggle to tell correct assertions from wrong ones when code is buggy (preprint). [arXiv 2410.21136](https://arxiv.org/html/2410.21136v1). A July 2026 preprint on ten bugs found oracles derived from written requirements aligned better with intent than with the buggy implementation, but not consistently. [arXiv 2607.10277](https://arxiv.org/pdf/2607.10277). Across 20,500 LLM-generated suites, Magic Number and Assertion Roulette were the most common smells (preprint). [Ouédraogo et al.](https://arxiv.org/abs/2410.10628v2). Assertion-free generated tests reach high coverage while checking nothing. [TestPilot](https://arxiv.org/pdf/2302.06527). Without an existing suite to imitate, 92.45% of Copilot's Python tests were failing, broken or empty. [AST 2024](https://conf.researchr.org/details/ast-2024/ast-2024-papers/2/Using-GitHub-Copilot-for-Test-Generation-in-Python-An-Empirical-Study).

In ImpossibleBench, where tests were altered to conflict with the spec, models deleted or edited tests or special-cased test inputs. A strict instruction to stop when tests look flawed cut one model's cheating from about 92% to about 1%; read-only tests also helped. [ImpossibleBench](https://openreview.net/pdf/f0edc0ee21784f063e1f7970ecbd360037762bb7.pdf). Anthropic's prompting guidance states that tests verify correctness rather than define the solution. [Claude docs](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-4-best-practices).

**Recommendation:** write expected values from the issue, docs, spec or user scenario before running the code. While implementing, treat existing tests as read-only; report a test that looks wrong instead of editing, deleting or special-casing it. Imitate the project's existing harness rather than generating tests in a vacuum.

## 15. LLM audits: structural smells are judgeable, semantic weakness needs execution

LLM test-smell detection reaches about 74–80% accuracy overall, higher for syntactic smells such as Assertion Roulette, and models sometimes introduce new smells while refactoring (preprint). [Santana et al.](https://arxiv.org/pdf/2506.07594). No study measured how precisely LLMs identify redundant tests or weak oracles, so mutation and pseudo-tested analysis remain the ground truth for those claims. Claude Code guidance recommends a reviewer in a fresh context not swayed by the generating reasoning, and warns that a reviewer asked to find problems will always find something; applying every finding yields tests for impossible cases. [Claude Code best practices](https://code.claude.com/docs/en/best-practices). Google's review guide asks whether tests would fail when code breaks and whether they would raise false positives when code changes underneath them. [eng-practices](https://google.github.io/eng-practices/review/reviewer/looking-for.html).

**Recommendation:** audit in a context separate from generation. Back "cannot fail" or "redundant" with a surviving mutant or replaced body; structural findings (no assertion, mock-call-only assertions, oversized snapshots, logic computing expectations, hidden setup values) can rest on judgment. Skip style nits. See [auditing](auditing.md).

## 16. Most deleted tests are obsolete; redundancy needs mutation evidence

In 24,431 confirmed test deletions across 7 Java projects, 91.4% were obsolete because their production subject was removed, and 7% were redundant. Deleting even 20% of tests developers judged redundant still lowered coverage or mutation score, and reduction tools found at most 54% of them. [MSR 2025](https://2025.msrconf.org/details/msr-2025-technical-papers/24/Understanding-Test-Deletion-in-Java-Applications). Large companies respond to tests that never fail by running them less often or later, not by deleting them. [Memon et al.](https://web.eecs.umich.edu/~xwangsd/courses/w23/readings/googletest.pdf); [Herzig et al.](https://microsoft.com/en-us/research/wp-content/uploads/2015/05/The-Art-of-Testing-Less-without-Sacrificing-Quality.pdf). A pilot study found 48.7% of bug-fixing changes conflicted with existing tests, meaning those tests had pinned the bug; it is a small pilot. [ESEM 2015](https://www.eecs.yorku.ca/~wangsong/papers/esem15.pdf). Characterization tests describe actual rather than intended behavior (secondary source). [Wikipedia](https://en.wikipedia.org/wiki/Characterization_test).

**Recommendation:** delete obsolete tests with their subject; call a test redundant only when it catches no fault another test misses; label pins of known-wrong behavior and flip them with the fix. "Never fails" is not deletion evidence.

## 17. Flakiness: keep flaky tests running, route them to owners, never filter results

Google's TAP saw 5.5M affected test targets in a month; only 63K ever failed, and only 1.23% found a real breakage or fix once flakes were removed. [Memon et al. 2017](https://web.eecs.umich.edu/~xwangsd/courses/w23/readings/googletest.pdf). Tests begin to lose value as flakiness approaches 1%, and reruns only defer the root cause. [SWE at Google ch. 11](https://abseil.io/resources/swe-book/html/ch11.html). Flakiness rises with test size (Meta: well below 1% for unit frameworks, up to 10% for some E2E). [Meta 2020](https://engineering.fb.com/2020/12/10/developer-tools/probabilistic-flakiness/). A per-size breakdown attributed to Google was seen only in a search snippet; treat it as indicative. [Google Testing Blog](https://testing.googleblog.com/2017/04/). Asynchronous calls were the leading cause across six Microsoft projects. [Lam et al. 2020](https://cs.gmu.edu/~winglam/publications/2020/LamETAL20FaTB.pdf). About 1 in 6 of Google's newly flaky tests exposed a production bug. [Google Testing Blog 2017](https://testing.googleblog.com/2017/04/where-do-our-flaky-tests-come-from.html). Flakes also corrupt history-based techniques: Meta's selection recall fell from about 0.9 to 0.7 once flakes were removed from evaluation data. [Machalica et al.](https://ar5iv.arxiv.org/html/1810.05286).

Working systems share one design: detect by retry flips or history, keep quarantined tests running off the blocking path, ticket an owner automatically, and re-enable when healthy. [Uber 2024](https://www.uber.com/en-US/blog/flaky-tests-overhaul/); [Atlassian 2025](https://www.atlassian.com/blog/atlassian-engineering/taming-test-flakiness-how-we-built-a-scalable-tool-to-detect-and-manage-flaky-tests); [Microsoft 2022](https://devblogs.microsoft.com/engineering-at-microsoft/improving-developer-productivity-via-flaky-test-management/). Slack's first approach filtered failing results in place; it leaked real failures and blocked merges when the flakiness backend was down, and was rolled back. Its replacement's disabled tests tended to stay disabled. [Slack 2022](https://slack.engineering/handling-flaky-tests-at-scale-auto-detection-suppression). Fowler advises capping quarantine size and age, since quarantined tests are useless as regression tests. [Fowler 2011](https://martinfowler.com/articles/nonDeterminism.html). Visibility alone helped Spotify (6% to 4% in two months). [Spotify 2019](https://engineering.atspotify.com/2019/11/test-flakiness-methods-for-identifying-and-dealing-with-flaky-tests/).

**Recommendation:** rerun new tests on correct source before accepting them; investigate missing waits first when one flips. Never add retries or skips to hide a flaky test; treat it as a possible production bug. Quarantine design is in [suite practices](suite-practices.md).

## 18. Doubles, test shape and snapshots at scale

Google's preference order is real implementation, then fake, then stub, with interaction verification avoided when possible. The team owning the real implementation owns the fake, places it at the dependency root, makes unsupported paths fail fast, and runs one contract suite against real and fake; `@DoNotMock` lets owners forbid mocks of types that have a good fake. [SWE at Google ch. 13](https://abseil.io/resources/swe-book/html/ch13.html); [Testing Blog 2024](https://testing.googleblog.com/2024/02/increase-test-fidelity-by-avoiding-mocks.html). Developers mock infrastructure, avoid mocking domain classes, and cite keeping mocks compatible as the main cost. [Spadini et al., EMSE 2019](https://springerlink.bibliotecabuap.elogim.com/content/pdf/10.1007/s10664-018-9663-0.pdf). No quantitative study links mock density to escaped bugs; the case rests on design argument and surveys. Spotify's honeycomb runs most service tests against a real DB. [Spotify 2018](https://engineering.atspotify.com/2018/01/testing-of-microservices). Fowler reads Google's "unit" tests as sociable and recommends tests that fail only for useful reasons over ratio debates. [Fowler 2021](https://martinfowler.com/articles/2021-test-shapes.html). Google's chapter 12 asks for tests through public APIs, state assertions, one behavior per test named for its outcome, no logic in tests, DAMP over DRY with builders whose defaults keep asserted values visible, and tests that change only when requirements change. [SWE at Google ch. 12](https://abseil.io/resources/swe-book/html/ch12.html). For components, Vue Test Utils prefers `mount` over `shallowMount` ("the more you stub, the less production-like your test becomes") and focuses tests on inputs (props, user interaction) and outputs (rendered DOM, emitted events). [VTU](https://test-utils.vuejs.org/guide/advanced/stubs-shallow-mount). Testing Library ranks role queries first and test ids last. [Testing Library](https://testing-library.com/docs/queries/about/). The testing trophy mocks only boundaries such as HTTP in its central integration tier. [Dodds 2021](https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications).

**Tensions:** Google and Spotify prefer real DBs and fakes, and the report's playbook suggests doubling the clock in a unit tier. The skill keeps its user-specified policy: DB/external-service mocks are allowed without real infrastructure, and replacing time or scheduling still needs the complex-algorithm exception. The research supports that policy's companion rules: label the proof boundary, keep owned logic real, and validate shared fakes.

Large snapshots go unread and get regenerated rather than reviewed. [Dodds 2017](https://kentcdodds.com/blog/effective-snapshot-testing). Vitest does not write snapshots when `CI` is set and offers inline snapshots. [Vitest](https://vitest.dev/guide/snapshot). insta can fail CI on orphaned snapshots and supports per-change review. [insta CLI](https://insta.rs/docs/cli/).

**Recommendation:** prefer small or inline snapshots, and accept a snapshot update only with a stated reason the new output is correct. Fake ownership, contract suites and snapshot governance are in [suite practices](suite-practices.md).

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
10. Which fault does this test catch that no existing test catches, and does it pass on reruns against correct code?
11. Would killing this survivor require a change-detector test?

## Remaining questions for deeper empirical work

- Optimal mix of targeted manual probes and automated mutation tools for small agent changes.
- How well mutation results generalize to authorization, async/lifecycle, and distributed defects across stacks.
- Cost and effectiveness of maintaining fake/provider contract parity in smaller projects.
- How to measure scenario contribution without incentivizing duplicate tests or artificial count targets.
- How independently generated expectations affect agent-test oracle quality. One small 2026 preprint (section 14) suggests requirement-derived oracles help but are not consistently reliable.
- How precisely LLM auditors identify redundant tests or weak oracles; no study measured it (section 15).
- Quarantine caps, deflake deadlines and tier ratios; no public source gives numbers (section 17).

No universal answer is established here. Avoid turning these open questions into hard thresholds or claiming causal effectiveness beyond the cited evidence.
