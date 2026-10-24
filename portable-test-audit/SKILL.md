---
name: portable-test-audit
description: Audit and consolidate existing unit and integration tests after TDD. Remove redundant, brittle, and implementation-coupled tests while preserving meaningful behavioral contracts and regression protection. Use for requests to prune, deslop, simplify, deduplicate, or maintain a bloated test suite.
---

# Portable Test Audit

Turn an exploratory or AI-generated test suite into a small, trustworthy **permanent regression suite**. Tests written during TDD may have been valuable while shaping the implementation without all deserving permanent maintenance. The objective is **maximum distinct behavioral protection per unit of complexity**, not the fewest tests or the highest coverage percentage.

This skill is self-contained: use only the repository's existing language, commands, test framework, and tooling. Do not depend on another skill, external reference file, repository layout, new package, or custom script. Apply the same method to unit, integration, and end-to-end tests where in scope.

## Non-negotiable rules

- **Behavior before structure.** Prefer contracts observable by a consumer, caller, external system, or meaningful subsystem boundary. Do not pin private call counts, helper names, import structure, or incidental intermediate values unless they are themselves contractual.
- **A keeper must actually detect the bug.** Neither shared line coverage nor the mere presence of another test proves equivalent protection. Identify a remaining assertion that would fail for the same plausible defect.
- **Do not trade correctness for a quota.** Never invent a deletion target. If the user specifies one, treat it as a goal subordinate to evidence. Stopping short is preferable to removing an independent guard.
- **Retain valuable distinctions.** Similar-looking tests can protect different branches, permissions, providers, integration layers, configurations, failure modes, or historical regressions. Do not collapse them just because they execute the same code.
- **Honor existing controls.** Do not change coverage thresholds, CI gates, production behavior, or test-runner configuration to make pruning appear successful. Do not commit, push, or create a PR unless requested.
- **Be truthful about verification.** A skipped, unavailable, or failing validation step must be reported, never described as passed.

## 1. Scope and mode

Infer scope from the request: specified files or feature, a subsystem, or the full repository. If no scope was given, start with the relevant current feature/test directory and expand to overlapping tests elsewhere. Do not ignore cross-directory duplicates that enforce the same contract.

- **Audit/report:** For requests to *review*, *assess*, or *identify*, produce actionable findings without modifying files.
- **Consolidate/apply:** For requests to *clean up*, *remove*, *prune*, or *consolidate*, edit the working tree and verify the result.
- **Campaign:** For repository-wide scope or an explicit quantitative target, group work by production ownership boundary and apply the workflow in reviewable batches. Maintain a brief ledger; no separate artifact is required.

Check `git status` and existing test instructions first. Preserve unrelated local changes. If there is no version control, take extra care to preserve the original content before broad edits.

## 2. Establish a reliable baseline

Before deleting anything:

1. Discover the repository's documented commands and actual test framework. Use its normal non-watch test command; stop active watchers during edits.
2. Run in-scope tests and record passing, failing, skipped, and conditionally skipped cases. A failing test is not disposable just because it fails: determine whether it represents an existing product defect or a broken test.
3. If coverage is already configured, record line and branch coverage for the scope **and the touched production files**, not only an aggregate percentage. Preserve a baseline report that later runs cannot overwrite.
4. Record a test count (ideally from the runner), approximate test/support code size, and execution time when easy to measure. These are descriptive metrics, not success criteria.
5. Note existing minimum coverage/quality gates. Keep them intact.

Use the project's scripts by preference. Example tooling **only if already available**:

| Ecosystem | Possible existing commands |
| --- | --- |
| Vitest | `vitest run`, `vitest run --coverage`, `vitest list --json` |
| Jest | `jest --runInBand --watch=false`, `jest --coverage --watch=false` |
| pytest | `pytest -q`, `pytest --collect-only -q`, `pytest --cov=<package> --cov-branch --cov-report=json` |
| Go | `go test ./...`, `go test ./... -coverprofile=coverage.out`, `go tool cover -func=coverage.out` |
| Other | Use the repository's own test/coverage equivalents; do not install a tool solely for this audit. |

Where measurement is unavailable, proceed by contract mapping and runner results; state the missing baseline explicitly. Do not guess counts or deltas.

## 3. Build a contract map

Review **the complete test**, its actual assertions, the production entry point, relevant callers/callees, neighboring tests, fixtures and mocks, and history or bug references when available. A descriptive test name is not evidence of what it protects.

Group tests by *behavioral guarantee*, not by filenames or test-framework syntax. For each group, determine:

- What user-visible, public-interface, persistence, or subsystem contract is protected?
- What plausible regression makes this test fail? Would a harmless implementation refactor also fail it?
- Which test gives the clearest, most realistic protection of that contract, and at what layer?
- Does another test enforce the **same failure mode**, or only touch the same lines?
- Is the setup or mock providing the answer that the implementation should have produced?
- Does a seam/export/helper exist only for this test? Verify all **non-test** consumers before suggesting its removal.

Classify candidates internally or in a compact ledger:

| Mark | Meaning | Action |
| --- | --- | --- |
| KEEP | Distinct, credible, independently checked contract | Retain; move to the right owning suite if misplaced |
| MERGE | Good contract duplicated or fragmented across tests | Transfer any unique assertion/case into a named keeper, then remove the duplicate |
| REPAIR | Worthwhile contract, but the assertion/mocking is weak or misleading | Correct the test without weakening its guarantee |
| DROP | No meaningful contract, or stronger equivalent proof remains | Delete after documenting the reason and keeper, if applicable |
| INVESTIGATE | Ambiguous, skipped, or baseline-failing test | Preserve until its purpose or failure is understood |

For each proposed MERGE or DROP, name the exact remaining keeper (or explain precisely why there is no legitimate contract). If you cannot, do not remove it.

## 4. Look for low-value test patterns

These patterns prompt inspection; they are **not automatic deletion rules**:

- An assertion-free smoke run, tautology, expected value calculated by the code under test, or assertion that only confirms a mock's canned output.
- Multiple tests of the same contract differing only in incidental fixtures, names, formatting, or setup; duplicated checks across files or layers without a distinct failure mode.
- Expectations about private helper invocation, internal call order, count, exact source text, import locations, or temporary implementation choices. **Exception:** order or exact bytes may be observable protocol behavior.
- Huge snapshots or inventories of test-authored data that mainly mirror the fixture. **Exception:** approved output snapshots and exact compatibility formats can be real contracts.
- Excessive per-provider replays of a single shared algorithm where the provider-specific adapter already has focused checks. **Exception:** independent implementations or provider contracts merit independent tests.
- An isolated mock-heavy unit test duplicating a stronger realistic integration test, where the unit test contributes no unique logic, error path, or diagnostic benefit.
- One test per tiny TDD iteration even though the final behavior is better expressed as a table of meaningful equivalence classes and boundaries.
- Negatives that pass due to the wrong guard, tests that assert a weaker condition than their title, and disabled tests without an understood rationale. **Repair or investigate** these before considering deletion.
- Test-only exports, reset functions, wrapper objects, global switches, or production indirection whose sole remaining purpose is to support discarded tests. Remove only after confirming zero legitimate production callers.

Prefer **a few focused tests for significant partitions** (normal case, relevant boundary, invalid input, failure/recovery, security or concurrency where applicable) over a combinatorial inventory of today's internal branches. Parameterize related cases when it improves clarity, but avoid one giant test where failures are hard to diagnose.

## 5. Protection that must survive

Err toward keeping any independent guard for:

- Public APIs, CLI behavior, user-facing output, serialization/protocol details, persisted data, schemas, migrations, compatibility, packaging, configuration, defaults, platform differences, or cross-language boundaries.
- Authorization, isolation, secrets, validation, error handling, retries, idempotency, concurrency, ordering, and transaction boundaries where failures have meaningful consequences.
- Historical bugs and credible edge cases with a distinct failure mechanism, even when line coverage overlaps with a happy-path test.
- Real integrations and contract tests that detect wiring/configuration defects missed by isolated unit tests. Preserve multiple layers when they catch **different** kinds of failures.
- Important invariants across input classes; exact expected outputs when they are contractual, not incidental.

A slow test, snapshot, source-level assertion, or duplicated layer can still be justified. Judge it by the failure it uniquely catches, not by its appearance. Treat skipped tests as investigation items: determine whether the skip is environmental, deliberate, stale, or hiding a bug; do not count deleting or moving them as a quality gain.

## 6. Consolidate incrementally

For each production owner or coherent group:

1. Choose the strongest keeper(s) first. Prefer realistic behavior at a stable boundary with controlled external dependencies, plus smaller unit tests for independent complex logic. Avoid replacing many checks with one fragile end-to-end test.
2. Move any **unique** assertions, boundary rows, or regression cases into the keeper **before** deleting duplicates. Keep the case's ability to fail for the correct reason.
3. Remove redundant declarations, over-specific fixture data, useless snapshots, and unused test helpers. Only remove production test seams or wrappers when proven unused by real callers; don't alter application behavior in a testing cleanup.
4. Re-run the changed test files plus adjacent owner/integration suites. Inspect the diff for accidental weakening of assertions and unintentional production edits.
5. Repeat while defensible consolidations remain. No required percentage or count reduction; don't keep pruning merely to improve a metric.

For a large campaign, repeat steps 3–6 across ownership groups, and keep a summary of KEEP/MERGE/REPAIR/DROP/INVESTIGATE decisions with owner and named keeper. Review whether an entire redundant testing *layer* can be removed, rather than selecting tests one by one at random.

## 7. Verify behavior survived the pruning

- Run the full relevant suite after all batches; compare failures and skips to baseline. If a test failed before, disclose that rather than claiming success.
- Re-run the **same** configured coverage command, when available. Inspect line **and branch** deltas globally and on affected production files. A drop needs explanation and contract-level justification, even if existing thresholds still pass. Do not loosen thresholds or make unrelated tests inflate the result.
- Coverage alone is weak evidence: when a removed test held the only meaningful assertion for a contract, make sure its keeper asserts the same failure. For uncertain high-value cases, perform a **small, reversible mutation** (e.g., flip the relevant condition or omit a crucial persistence action) and verify that a retained test fails. Restore the original source completely; never overwrite unrelated local work. Use an existing mutation-testing tool if the project has one, but do not introduce one as a dependency.
- Check relevant lint/type checks if test restructuring affects them; check CI references and documentation for renamed or deleted tests.
- Review `git diff` (and `git diff --numstat` if available). Distinguish test/support line changes from any production line changes. No commit or push without an explicit request.

If verification is inconclusive, keep the questionable test or report it as unverified instead of claiming the suite is fully protected.

## 8. Final report

Provide a compact summary with:

1. **Scope and result:** files or subsystem audited; which suites were merged, removed, repaired, or retained.
2. **Before / after:** test count, test/support lines, execution time, and line/branch coverage **only where actually measured**; affected-file coverage changes when relevant.
3. **Contract preservation:** for nontrivial deletions, identify the remaining keeper(s) and important behaviors retained; call out any deliberate multi-layer overlap.
4. **Validation:** exact commands and their outcomes, baseline failures, limitations, and any mutation spot checks.
5. **Residual risks:** ambiguous/skipped cases, suspected product bugs, or valuable tests left in place with rationale.
6. **Prevention:** one short repository-agnostic rule suitable for future TDD agents, based on the actual redundancy found (for example, "Before adding a permanent test, identify which distinct externally observable failure it catches that existing tests would miss").

Do not overstate test quality from deletion counts or coverage alone. The correct result may be a modest reduction if the existing suite already protects distinct behaviors.

---

## Attribution and license

This standalone adaptation is informed by Matthew Blode's `test-audit` skill in `mblode/agent-skills`, which credits OpenClaw's `test-audit`. It consolidates the original skill's audit, campaign, and coverage guidance into one file and changes the default from quota-driven pruning to evidence-driven maintenance. The original repository is MIT licensed. The following notice is included for attribution and license compliance; it is **not** an operational dependency.

MIT License

Copyright (c) 2026 Matthew Blode

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
