# Targeted Fault Probes

Use when running isolated mutations, investigating false-green tests, or verifying suspected production bugs during an audit. Adapt to existing runner; these checks do not require a mutation framework or exhaustive campaign.

## Two kinds of probe

- **Bug probe:** asserts the correct contract against unmodified source and fails because the defect exists. Use to verify a suspected production bug, including every bug claim in an audit.
- **Sensitivity probe:** runs a test against an isolated copy of source carrying one plausible defect; the test must fail. Use to show that an existing or proposed test catches the defect.

## Probe contract

- Assert the **correct** contract from the product/API, never the current output. A probe that encodes observed behavior proves nothing.
- Include a **passing control** in the same file: same harness, same input shape, a value the code handles correctly. It proves setup works and that the failing case fails because of the defect. A control that fails means the probe is wrong, not the code.
- Count only **assertion failures** tied to the claim. Import/setup errors, TypeErrors from probe code, and timeouts are not evidence. An error thrown by production code counts when the claim is "throws".
- Use the most real entry practical: the project's own harness, real components/stores, allowed DB/service doubles only. When the full entry is impractical, probe the narrowest real function that owns the defect and name the user step left inferred.
- Give probe files a distinct suffix (for example `*.probe.test.ts`) and a header comment naming the claim or bug id. Record the command and the decisive output line verbatim (expected vs received).
- Rerun the correct-source control several times (TestGen-LLM used five). A probe or new test that flips on unchanged source is not evidence; fix its waits or state first.
- Batch probes into few runs when environment boot is slow. Do not stop processes you did not start.

## Run probes outside the checkout

Keep probes in a scratch folder and reuse the project's own runner configuration, so aliases, environment, setup files and transforms match without adding files to the repo. Check `git status` and default cache folders before and after the first run; redirect any write that lands in the checkout or shared dependencies.

Vitest (also covers framework environments built on vite). Import production code by absolute path or project alias; rewrite relative imports in copied files to absolute paths.

```js
// <scratch>/harness/vitest.probe.mjs
import { join } from 'node:path'
import { pathToFileURL } from 'node:url'

export default async (env) => {
  const root = process.env.PROBE_ROOT // package folder holding vitest.config.ts
  const dir = process.env.PROBE_DIR // scratch probe folder
  if (!root || !dir) throw new Error('Set PROBE_ROOT and PROBE_DIR')
  const mod = await import(pathToFileURL(join(root, 'vitest.config.ts')).href)
  const config = typeof mod.default === 'function' ? await mod.default(env) : await mod.default
  config.root = root
  config.cacheDir = join(dir, '.vite-cache') // runner cache defaults to node_modules/.vite
  // Let the dev server read probe files outside the project root.
  config.server = { ...config.server, fs: { ...config.server?.fs, strict: false, allow: ['/'] } }
  config.test = {
    ...config.test,
    dir,
    include: ['**/*.probe.{test,spec}.ts'],
    coverage: { ...config.test?.coverage, enabled: false },
    typecheck: { enabled: false },
  }
  // Sensitivity probe: redirect one resolved module to its mutated scratch copy.
  const { MUTANT_FROM: from, MUTANT_TO: to } = process.env
  if (from && to) {
    config.plugins = [...(config.plugins ?? []), {
      name: 'probe-mutant',
      enforce: 'pre',
      async resolveId(source, importer, options) {
        const resolved = await this.resolve(source, importer, { ...options, skipSelf: true })
        return resolved?.id === from ? to : null
      },
    }]
  }
  return config
}
```

Run from the package folder: `PROBE_ROOT=$PWD PROBE_DIR=<scratch>/probes/<group> npx vitest run --config <scratch>/harness/vitest.probe.mjs`. Add `MUTANT_FROM=<absolute source path> MUTANT_TO=<scratch mutant>` for a sensitivity run; the same file must pass without them. Adjust for a config that lives elsewhere or defines several projects.

Other runners, same idea (confirm on first run):

- **Jest:** a scratch config spreading the project config with `rootDir` kept at the project, `roots: ['<scratch>/probes']`, a probe `testMatch`, `cacheDirectory` under scratch, coverage off. Map a mutated module with `moduleNameMapper`. If probe files are not transformed, point the transformer at the project's babel/TS config explicitly.
- **pytest:** `PYTHONDONTWRITEBYTECODE=1 pytest -p no:cacheprovider --rootdir <repo> -c <repo config> <scratch>/probes`. Fixtures in the repo's `conftest.py` are not discovered for files outside the tree; load that conftest as a plugin with `-p <module>` and the repo on `PYTHONPATH`.
- **Cargo:** a scratch crate with a `path` dependency on the crate under test and integration tests in its `tests/`, a copy of the workspace `Cargo.lock`, and a build directory outside the checkout unless the project mandates its own build wrapper. This reaches public API only; private items need a scratch copy of the crate.

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

When no claim-specific defect is obvious, start by deleting the changed statement block; at Google this bluntest operator accounted for most mutants coupled to high-priority bugs. One probe per changed line is usually enough.

A defect that violates no promised observable behavior is equivalent for this contract. Do not force detection by asserting private implementation structure. Judge equivalence from code and contract, not code comments, which can mislead.

Before writing a test to kill a survivor, ask whether that test would be a change-detector: it breaks on harmless refactors without catching a user-visible bug. Survivors in logging, metrics, timeouts/retry/backoff, feature-flag plumbing, collection capacities and debug formatting usually are. Record those as not worth a test with a one-line reason instead of pinning the constant. A timeout or flag stays testable when its observable behavior is itself the contract.

Verify reachability before treating a survivor as weak coverage. An application may protect its caller by cloning while an exported helper still corrupts its own argument or a later hook receives damaged transport data. Those are separate contracts: use a real supported consumer/extension or a focused exported-function test, without mocking the protective collaborator. Conversely, moving an internal abort notification can be harmless when the real dispatcher still awaits work and propagates errors; an observer seeing different timing does not itself establish a user defect.

## Probe both defects and harmless variation when justified

For strong request/data assertions, distinguish required ordering from incidental order. A filter conjunction or unordered key set may permit reordering; preserve all predicates/values and multiplicity rules without requiring original indexes. A small equivalent-order probe can expose overcoupling after tightening assertions. Do not mandate this probe for unrelated edits.

For actual bugs, first demonstrate the unmodified source failing the intended contract. If production fixes are outside the task scope, retain the isolated regression and report the bug instead of weakening the assertion or silently expanding scope.

When replacing weak tests, compare their actual inputs and observables before retiring them. For example, a new batch scenario does not cover a retired single-result normalization case. Exercise the distinct shape through the real public consumer, and use a targeted normalization fault if overlap is uncertain.

## Resource probes must leave bounded diagnostics

For a lifecycle probe, separate an explicit final-release assertion or native blocked-event error from a bounded contract poll and from the runner timeout. A timeout alone may show the suite rejects a leak, but is weaker evidence and can hide cleanup hangs. Keep acquired handles in failure-safe teardown; settle/cancel held external work without awaiting the broken completion indefinitely. Do not repair production release during the scored observation.

Record exact final source/test bytes or an equivalent immutable snapshot for scored probes. Re-run affected probes after meaningful changes: a previously effective deletion can become harmless when another path now supplies the same protection. Do not present an earlier mutation report as proof of different final code.

Teardown must cover setup failures after any acquisition and must not depend solely on the same discovery or termination helper being faulted. Preserve independently recorded handles and bounded fallback cleanup. Distinguish a named completion/rejection assertion from setup errors and whole-runner timeouts. For safe harness execution and fresh skill trials, use [execution and evaluation](execution-and-evaluation.md).
