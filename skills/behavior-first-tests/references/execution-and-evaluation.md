# Execution and Independent Evaluation

Read execution guidance when tests own browsers, subprocesses, or other platform resources. Read evaluation guidance when assessing a changed skill or comparing agent approaches. This does not mandate browser infrastructure, agent delegation, or a benchmark for ordinary tests.

## Keep execution within the task's constraints

Prefer headless browser execution. When native focus or another documented platform contract requires headed execution, use an isolated display/session or equivalent supported platform facility. Do not route automation onto the user's desktop or existing browser session contrary to their instructions. If that isolation is unavailable, finish unaffected work and report the missing native proof; do not silently fall back to desktop windows.

Check inherited remote-browser routing before startup. A headless flag or local display setting does not constrain a connection to an already-running remote browser. Prefer rejecting unsafe routing over attempting to clean up someone else's session. Keep specific environment names and platform commands in the project's harness; a generic testing skill should not require Linux or Xvfb.

Own created resources explicitly. Signal or close only those resources; never kill all browsers/processes by name. Descendants may detach into different process groups, so a parent/group exit alone need not prove release. Use supported ownership handles/markers and bounded shutdown, including forced cleanup when appropriate. Verify actual command interruption as well as a cleanup helper when command wiring is part of the claim.

Protect acquisition itself: register teardown before later setup can fail, keep independently acquired identities, and ensure emergency cleanup still works when the function under test is defective. Separate the scored release observation from fallback cleanup. Report which resources stopped and which remain unverified.

Fault copies need private writable caches, generated modules, output and temporary paths as described in [fault probes](fault-probes.md). Shared dependency links alone do not provide that isolation.

## Evaluate revised guidance without contaminating the trial

When a realistic trial is warranted and authorized:

1. Freeze an untouched source/test fixture and the candidate skill. Keep expected defects and scoring notes outside the agent's workspace. Record a snapshot identifier or hashes for reproducibility.
2. Give an independent agent the candidate skill, a realistic user request, supported contracts, raw source/current tests and execution constraints. Withhold suspected gaps, answer tests, mutant diffs, prior findings and author conversation. If independence cannot be established, call the result a guided check.
3. Start each retry from the same untouched fixture or a new comparable fixture. Never hand the next agent tests improved by a previous attempt and call it a fresh evaluation.
4. Review the resulting tests, then run the correct implementation and independently selected, legitimate source faults against those exact final tests. Score meaningful assertion failures and useful scenarios; exclude equivalent changes, setup failures and unrelated timeouts. Check allowed mock boundaries and failure-safe cleanup too.
5. Make only changes justified by observed behavior. Preserve failed attempts and outcome boundaries in temporary evidence; do not turn a small trial into a universal quality claim.

Evaluate a skill revision with application outcomes, not checks that its prose contains prescribed phrases. Use the project's existing runner and allowed external doubles; do not add live services or a new test stack solely for the trial.
