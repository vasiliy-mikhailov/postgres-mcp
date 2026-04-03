---
name: python-dependency-quick-remediation
description: Search deeply for Python dependency vulnerability remediations, keep a full SEARCH_LEDGER, and preserve the final accepted result in a feature branch in the current repository.
compatibility: opencode
---

# Python dependency quick remediation

Use this skill only for dependency vulnerability remediation in Python repositories.

## Goal

Find the best **partial** dependency-security improvement while preserving a full search ledger.

A valid result means:
- materially improves the Trivy result
- does not introduce new vulnerabilities
- does not cause harmful downgrades
- passes dependency resolution
- passes verification
- does **not** worsen the baseline pytest test signature
- does **not** require fixing every remaining vulnerability

Do **not** optimize for “zero vulnerabilities at any cost”.
Do **not** modify production source code.
Do **not** add new direct or dev dependencies just to force transitive upgrades.

## Definitions

- A **MOVE** is one concrete dependency-resolution action.
- A **CANDIDATE** is a **combination of MOVES**.
- **DEPTH** is the number of MOVES in the candidate.
- **DRILL-DOWN** means selecting a candidate for expansion into deeper combinations.

## Allowed move types

Prefer these, in this order:

1. package upgrade via `uv lock --upgrade-package <package>`
2. constraint set via `[tool.uv].constraint-dependencies`
3. override via `[tool.uv].override-dependencies` only if no safer move works

## Forbidden move types

Reject any candidate that:
- adds transitive vulnerable packages as new direct dependencies
- adds transitive vulnerable packages as new dev dependencies
- modifies production source code
- insists on fixing every remaining vulnerability before accepting a result

## Test-signature rule

Define `TEST_SIGNATURE` as:
- collected
- passed
- failed
- errors
- skipped
- xfailed
- xpassed

First measure the baseline `TEST_SIGNATURE` on the clean repository.

For any candidate selected for drill-down, compare its `TEST_SIGNATURE` against baseline.

A candidate is **REJECTED** if the test signature worsens in any way, including:
- `failed` increases
- `errors` increases
- `xfailed` increases
- `xpassed` changes unexpectedly
- `passed` decreases

Pytest exit code alone is **not sufficient**.

## Search policy

Search up to depth 10 unless the user asks otherwise.

Build candidates iteratively and preserve a full ledger of evaluated candidates.

Verification is mandatory **every time a candidate is selected for drill-down**.

A candidate may be expanded only if:
- dependency resolution succeeded
- Trivy improved materially or stayed policy-valid
- `uv sync` succeeded
- `pytest` completed
- candidate `TEST_SIGNATURE` matches baseline exactly or improves on it
- the lightest valid CLI smoke check succeeded

## Required evaluation loop

For every candidate evaluation:

1. Start from clean current repo state.
2. Apply the full move combination.
3. Run dependency resolution.
4. Run a fresh Trivy filesystem vulnerability scan.
5. If this candidate is selected for drill-down, run verification:
   - `uv sync`
   - `pytest`
   - capture `TEST_SIGNATURE`
   - run the lightest valid CLI smoke check for the project CLI entry point
   - if a repo-native test command exists, prefer it as an additional check
6. Record results into the search ledger.
7. Revert before the next candidate.

## Preservation and branch policy

All intermediate experiments must be reverted before the next candidate.

However, after the search finishes and the best candidate is classified as **ACCEPTED** or **VERIFIED_BUT_RISKY**, preserve it in the **current repository** as a dedicated feature branch.

Rules:
1. Do **not** use a separate worktree as the final preservation target.
2. Create or switch to a dedicated feature branch in the current repository.
3. Re-apply the winning move combination in that branch.
4. Re-run dependency resolution.
5. Re-run verification:
   - `uv sync`
   - `pytest`
   - capture `TEST_SIGNATURE`
   - smoke check
6. Confirm the final preserved state still satisfies acceptance rules.
7. Stage only intended dependency-related file changes.
8. Commit them.
9. Push the branch to `origin`.
10. If GitHub CLI (`gh`) is available, attempt to create a pull request.
11. If GitLab CLI (`glab`) is available, attempt to create a merge request.
12. If PR/MR creation is unavailable, still leave the branch pushed and report the next manual step.
13. Do **not** merge automatically unless the user explicitly asks.
14. Do **not** revert the final preserved branch state at the end.

### Branch naming

Prefer branch names like:
- `quickwin/deps`
- `quickwin/deps-<date>`
- `quickwin/<repo>-deps`

## Preferred commands

Choose commands appropriate for the repo, but for `uv`-based Python repos prefer:

- baseline sync: `uv sync`
- upgrade move: `uv lock --upgrade-package <package>`
- tests: `uv run pytest -q -rxX`
- smoke check: the lightest valid CLI invocation for the repo
- Trivy scan: filesystem vulnerability scan against the repo root

For final preservation prefer:

- create branch:
  - `git switch -c quickwin/deps`
- or switch if it already exists:
  - `git switch quickwin/deps`
- commit:
  - `git add pyproject.toml uv.lock`
  - `git commit -m "Quick-win dependency remediation"`
- push:
  - `git push -u origin quickwin/deps`

If GitHub CLI is available, prefer:
- `gh pr create --title "Quick-win dependency remediation" --body "<short summary>"`

If GitLab CLI is available, prefer:
- `glab mr create --fill`

If `pytest` reports no tests, say so explicitly.

## Output format

Return exactly one artifact:

SEARCH_LEDGER

BASELINE
  trivy_total:
  top_level_deps:
  baseline_test_signature:

CANDIDATES
  - candidate_id:
    depth:
    move_combination:
    score:
    checked:
    selected_for_drill_down:
    trivy_before:
    trivy_after:
    upgraded_nodes:
    downgraded_nodes:
    new_vulns:
    removed_vulns:
    lock_churn:
    sync_result:
    pytest_result:
    baseline_test_signature:
    candidate_test_signature:
    test_signature_match:
    smoke_result:
    expansion_allowed:
    verdict:

EXPANSION_ORDER

BEST_CURRENT_CANDIDATE
  candidate_id:
  depth:
  move_combination:
  score:
  sync_result:
  pytest_result:
  baseline_test_signature:
  candidate_test_signature:
  test_signature_match:
  smoke_result:
  final_classification:

PRESERVED_RESULT
  preserved: true|false
  preservation_mode: branch|none
  branch_name:
  pushed: true|false
  remote_name:
  pull_request_or_merge_request_created: true|false
  review_url:
  changed_files:

STOP_RULE

## Scoring priorities

- bigger vulnerability reduction is better
- passing verification is better than failing verification
- matching or improving baseline test signature is better than merely having a zero exit code
- fewer downgraded packages is better
- fewer new vulnerabilities is better
- lower lock churn is better
- smaller changed surface is better

## Classification priorities

- **ACCEPTED** = material security improvement and drill-down verification passed with no test-signature regression
- **VERIFIED_BUT_RISKY** = material security improvement, verification passed, no test-signature regression, but major upgrades or high churn are involved
- **REJECTED** = verification failed, test signature worsened, new vulnerabilities appeared, harmful downgrades appeared, or policy was violated

## Important

- Do not require fixing every remaining vulnerability.
- A partial but verified win is acceptable.
- Do not patch source code.
- Do not auto-merge.
- Revert all non-final experiments.
- Preserve the final accepted candidate in a dedicated feature branch in the current repository.
- Push that branch when possible.
- Attempt to create a PR/MR when the appropriate CLI/tooling is available.
- Do not revert the final preserved branch state at the end.
- Do not summarize outside the artifact.
