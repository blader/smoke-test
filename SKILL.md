---
name: smoke-test
description: "End-to-end branch smoke testing including uncommitted code. Use when asked to verify what a branch is supposed to do, create a concrete test plan, run local validation, and run remote validation via temporary PRs/CI (including multiple temporary branches when needed), with explicit feature-level behavior verification (not just app load/login) and screenshot evidence from the webapp showing the feature working, including clear state-change and persistence proof when applicable."
---

# Smoke Test

Validate the current branch end-to-end, including uncommitted state.

## Non-negotiables
- Include staged and unstaged local changes in the tested scope.
- Start with an explicit test plan and pass/fail criteria.
- Reproduce the target failure state on `origin/latest` (or user-specified baseline) before validating the candidate branch.
- Use the same inputs/data for baseline and candidate runs when reproducing failures.
- Write a Markdown results report for every smoke run.
- Use the canonical report path in the repo root:
  - `.smoke-results/<branch>/<timestamp>/smoke-results.md`
  - where `<branch>` is a filesystem-safe branch slug (for example `$(git branch --show-current | tr '/' '-')`)
  - and `<timestamp>` is UTC formatted as `%Y%m%d-%H%M%SZ`
- Update the Markdown report continuously during execution (plan, commands run, outcomes, evidence paths, and final verdict).
- Validate locally first, then remotely through temporary PRs and CI.
- Use temporary smoke branches for remote checks; avoid mutating the user's main working branch.
- Preserve local working state and avoid destructive git commands.
- Include PR context when available: title, description, linked issue/spec text, and current code review comments.
- Convert relevant open review comments into explicit smoke scenarios or assertions (not just narrative notes).
- Whenever feasible in the current environment, start the webapp and verify it loads successfully.
- When webapp verification is feasible, verify the login page successfully loads (or expected auth redirect occurs), not just a generic HTTP 200.
- Login/load checks are required but never sufficient for feature PRs.
- If the branch changes user-facing behavior, execute the actual feature workflow end-to-end in the running app using realistic inputs.
- Define scenario-level assertions before execution as concrete user-visible outcomes (what must change, where it must be visible, and what counts as pass/fail).
- Validate at least one primary happy-path scenario and one meaningful edge/regression scenario tied to the PR.
- For features where user input/configuration affects behavior, verify both:
  - the feature state/input itself (selected option, entered value, chosen mode, etc.); and
  - the downstream product behavior it is supposed to change (for example labels, values, ordering, calculations, visibility).
- If the feature is stateful (persisted config, saved edits, sticky UI state, etc.), verify both:
  - the immediate post-action effect; and
  - persistence after reload/re-open (or explicitly prove non-persistence if that is expected behavior).
- For persisted feature behavior, post-reload verification must include both the saved feature state/input and the affected downstream UI/output state.
- If the changed behavior can vary by runtime mode/flag/backend path, run a scenario matrix across relevant modes (not only the default mode).
- For backend changes that can differ across DS2DB vs VDS pathways, run and report both pathways when applicable to the touched code.
- Determine expected feature behavior from the actual code changes (and changed tests) before running smoke checks; do not guess expected outcomes.
- Capture screenshot evidence that proves behavior (before/after or equivalent decisive states), not just a screenshot of the login page.
- Required for feature PRs: at least one screenshot must show the real webapp UI while exercising the PR feature successfully (the success state visible in the product UI).
- Required for feature PRs: screenshot evidence must map to assertions and include both the feature input/action state and the resulting product state.
- Screenshot evidence must prove correctness of the feature outcome, not just that a menu opened or an input/action was clicked.
- Do not use loading/skeleton-only screenshots as proof of feature success.
- Do not treat generated HTML summaries, API-only response dumps, or screenshots of non-product artifact pages as sufficient feature evidence.
- For feature PRs, drive UI evidence via Playwright: use an existing Playwright test/harness when available, or run a direct Playwright script that performs the feature workflow in the webapp.
- If test execution is blocked, actively debug and fix the blocker first (env/config/startup/data/auth/flakes), then rerun.
- Only use fallback evidence after attempted blocker remediation paths are exhausted and documented with commands + outcomes.
- If webapp verification is still not feasible, explicitly record blocker + fallback evidence and do not silently skip it.

## Workflow

### 0) Initialize Markdown results file
1. Resolve repo root and define canonical results path before testing begins.
2. Build canonical path components:
   - `BRANCH_SLUG=$(git branch --show-current | tr '/' '-')`
   - `RUN_TS=$(date -u +%Y%m%d-%H%M%SZ)`
   - `RESULTS_DIR="<repo-root>/.smoke-results/${BRANCH_SLUG}/${RUN_TS}"`
   - `RESULTS_MD="${RESULTS_DIR}/smoke-results.md"`
3. Create `smoke-results.md` with:
   - Branch and baseline refs.
   - Timestamp, environment summary, and test owner.
   - Planned scenario matrix and pass/fail criteria.
4. Append updates after each major step instead of waiting until the end.
5. Ensure every reported assertion maps to concrete artifact paths in this file.

### 1) Baseline on latest first (failure repro)
1. Resolve baseline ref (`origin/latest` unless user specifies another baseline).
2. Create an isolated baseline workspace/worktree so current branch state remains untouched.
3. Run the failure repro scenarios on baseline first, using the same input artifacts planned for candidate validation.
4. Record exact commands and whether the failure reproduces.
5. If failure does not reproduce on baseline, call that out explicitly before proceeding to candidate validation.

### 2) Understand expected behavior
1. Capture branch context:
   - `git branch --show-current`
   - `git status --short`
   - `git diff --name-status`
   - `git diff --stat`
2. Identify changed subsystems (API, webapp, scripts, infra, etc.).
3. Infer expected behavior from:
   - Diff and changed tests.
   - PR title/description (if present).
   - PR review comments and requested follow-up checks (if present).
   - Nearby docs/config tied to changed files.
   - Relevant changed code paths that implement the feature.
4. Produce a concise "what this branch should do" statement before testing.

### 3) Build a test plan
Create a plan before running checks. Include:
1. Test matrix by behavior/scenario, with baseline (`origin/latest`) and candidate columns.
2. Explicit feature scenarios tied to PR intent (happy path + edge/regression path).
3. Concrete assertion list per scenario (observable UI/API facts that must be true).
4. Concrete evidence plan per scenario (which artifact proves each assertion).
5. Mode/flag/backend matrix when behavior may differ by runtime mode.
6. DS2DB/VDS pathway matrix when touched code can execute through both (or explicit rationale when one path is not applicable).
7. Review-comment-derived scenarios/assertions when PR feedback identifies risk areas.
8. Local checks per scenario.
9. Remote checks per scenario.
10. Expected result and failure signal for each scenario.
11. Branch strategy:
   - Single temporary smoke branch by default.
   - Multiple temporary branches when isolating independent risk areas gives clearer signal.

### 4) Local smoke testing
1. Run baseline local repro first on `origin/latest`.
2. Run candidate local validation second with identical scenarios/inputs.
3. For each run, execute targeted lint/build/test commands for touched areas first.
4. For each run, execute broader smoke checks for integration points touched by the branch.
5. If a test blocker appears, debug + remediate it immediately:
   - Verify service/process readiness, ports, env loading, auth cookies/secrets, and test data preconditions.
   - Apply minimal targeted fixes needed to unblock execution.
   - Re-run the blocked scenario after each fix and log evidence of resolution.
6. Attempt webapp validation whenever feasible (not only UI-only branches):
   - Start the webapp and required dependencies.
   - Verify the app loads (for example: reachable URL + no immediate runtime crash).
   - Verify the login page route loads successfully (`/login` or app-specific auth entry route) and confirm a login signal (login UI marker or expected auth redirect target).
   - Do not treat status code alone as sufficient webapp validation; require a login-page/auth signal.
   - Execute the PR feature workflow end-to-end in UI/API flows, not just page-load checks.
   - Exercise the feature flow through Playwright (existing test harness preferred; otherwise direct script) so steps are explicit and repeatable.
   - Assert feature outputs/states explicitly against expected PR behavior at each key step, not just at the end.
   - For workflows where user input/configuration drives behavior, assert both the feature state/input and the expected downstream UI/data change it should drive.
   - For each stateful scenario, include a persistence check after reload/re-open and assert the state is retained (or correctly reset, if expected).
   - For persisted feature behavior, after reload/re-open assert both the feature state/input and the downstream UI/data outputs remain correct.
   - Capture screenshots for decisive checkpoints (for example: pre-action state, selected input/action state, post-action result state, and post-reload state).
   - Ensure decisive screenshots are taken from the running webapp product UI while performing the actual feature steps.
   - Ensure screenshot sets include enough context to prove the feature’s effect (feature state/input and affected content, either in one frame or paired frames).
   - Before each decisive screenshot, wait for stable loaded UI (not skeleton/loading placeholders).
   - Capture at least one machine-checkable artifact per scenario when possible (for example extracted labels/values in JSON) alongside screenshots.
   - Name artifacts clearly (`baseline|candidate`, scenario, mode/flags) and save stable file paths.
   - If blocked, capture the blocker and run the strongest available substitute checks.
7. Record exact commands and outcomes for both baseline and candidate.

### 5) Remote smoke testing via temporary PRs
1. Create one or more temporary smoke branches that include current uncommitted behavior snapshot.
2. Push each branch and open draft PR(s) for CI validation.
3. Monitor CI checks to completion and collect failures with logs.
4. If signal is ambiguous, create additional temporary branches to isolate suspect changes and re-run CI.
5. Keep branch naming systematic, for example:
   - `smoke/<base-branch>/<timestamp>`
   - `smoke/<base-branch>/<timestamp>-<focus>`

### 6) Compare baseline vs candidate, then local vs remote results
1. Compare baseline (`origin/latest`) and candidate outcomes first:
   - Failure reproduced on baseline and candidate.
   - Failure reproduced on baseline only.
   - Failure reproduced on candidate only.
2. Then classify candidate failures:
   - Real regression introduced by branch.
   - Existing failure already present on baseline.
   - Existing flaky/unrelated failure.
   - Environment/config-only issue.
3. Confirm whether behavior matches expected branch intent from Step 2.
4. For DS2DB/VDS-applicable changes, compare pathway outcomes and call out any divergence.

### 7) Report and close out
Always provide:
1. Plan executed (scenarios + criteria, including baseline and candidate phases).
2. Baseline repro commands and outcomes.
3. Candidate local test commands and outcomes.
4. Remote PR links and CI outcomes.
5. Webapp verification evidence (or explicit blocker + fallback evidence), including login-page load/auth signal evidence.
6. Feature-workflow evidence summary with:
   - scenario -> assertion -> artifact mapping,
   - screenshot file paths and what each proves,
   - explicit persistence evidence when applicable,
   - mode/flag matrix outcomes when applicable,
   - DS2DB/VDS matrix outcomes (or explicit non-applicability rationale),
   - review-comment scenario outcomes,
   - blocker-remediation log (blocker -> fix attempted -> result),
   - at least one screenshot of the webapp showing successful PR-feature execution.
7. Final verdict: `pass`, `pass with caveats`, or `fail`.
8. Outstanding risks and next actions.
9. Markdown report path and confirmation that it contains the full scenario/assertion/artifact ledger.

## Remote execution notes
- Prefer `gh` CLI for PR and check management when available.
- If remote operations are blocked (auth/permissions/network), state the blocker clearly and provide exact commands the user can run manually.
- Do not delete temporary remote branches or close temporary PRs unless requested.

## Command snippets

```bash
git branch --show-current
git status --short
git diff --name-status
git diff --stat
```

```bash
# Optional PR context pull (when gh auth is available)
gh pr view --json title,body,comments,reviews
```

```bash
# Optional pathway reconnaissance for backend migrations/modes
rg -n "ds2db|vds|dataset-to-db|DATASET_SOURCE|migration mode" go webapp
```

```bash
# Example baseline worktree flow (run failure repro on latest first)
BASE_REF=origin/latest
BASE_WT="../$(basename "$PWD")-smoke-baseline-latest"
git fetch origin latest
git worktree add "$BASE_WT" "$BASE_REF"
# Run the same repro steps in $BASE_WT first, then run candidate steps in current workspace.
```

```bash
# Example remote smoke branch flow
SMOKE_BRANCH="smoke/$(git branch --show-current | tr '/' '-')/$(date +%Y%m%d-%H%M%S)"
git switch -c "$SMOKE_BRANCH"
git add -A
git commit -m "smoke: snapshot for remote validation"
git push -u origin "$SMOKE_BRANCH"
gh pr create --draft --fill --head "$SMOKE_BRANCH"
gh pr checks --watch
```

```bash
# Example local login-page verification (adjust route/text marker for your app)
WEB_PORT=${WEBAPP_PORT:-3000}
curl -sS -L "http://localhost:${WEB_PORT}/login" -o /tmp/smoke-login.html
rg -n "Log in|Sign in|Continue with" /tmp/smoke-login.html
```

```bash
# Example screenshot artifact naming convention
ART_DIR="/tmp/smoke-artifacts/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$ART_DIR"
# Save artifacts as:
#   $ART_DIR/baseline-happy-before.png
#   $ART_DIR/baseline-happy-after.png
#   $ART_DIR/candidate-happy-before.png
#   $ART_DIR/candidate-happy-after.png
```

```bash
# Example Markdown results file initialization
REPO_ROOT="$(git rev-parse --show-toplevel)"
BRANCH_SLUG="$(git branch --show-current | tr '/' '-')"
RUN_TS="$(date -u +%Y%m%d-%H%M%SZ)"
RESULTS_DIR="$REPO_ROOT/.smoke-results/$BRANCH_SLUG/$RUN_TS"
RESULTS_MD="$RESULTS_DIR/smoke-results.md"
mkdir -p "$RESULTS_DIR"
cat > "$RESULTS_MD" <<'EOF'
# Smoke Test Results
- Branch:
- Baseline:
- Started at:

## Test Plan
- Scenario:
  - Assertions:
  - Evidence plan:

## Baseline Results
- Commands:
- Outcomes:
- Artifacts:

## Candidate Results
- Commands:
- Outcomes:
- Artifacts:

## Remote CI Results
- PR links:
- CI outcomes:

## Final Verdict
- Verdict:
- Risks / next actions:
EOF
```
