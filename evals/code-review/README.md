# pw-code-review eval

Code-based grader for the `pw-code-review` skill, designed after Anthropic's
[Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents).

## Run

```bash
npm run eval:review:grader        # free, deterministic: validates the grader itself
npm run eval:review               # live: runs the skill via `claude -p` (costs tokens)
EVAL_TRIALS=3 npm run eval:review # 3 trials per task -> pass@k / pass^k
```

Optional env: `EVAL_TRIALS` (default 1), `EVAL_MODEL`, `CLAUDE_BIN`. Raw reports and grades from
live runs are saved to `results/` (gitignored); read them when a task fails.

## How it follows the guidance

| Guidance                           | Here                                                                                                                                              |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Grade outcomes, not paths          | `grader.ts` checks the report and the workspace afterwards. No tool-call order, no turn counts. Turns/time/cost are recorded, never graded.        |
| Avoid brittle checks               | Findings match on file + (cited `§N` **or** keyword) + line range. Bullet style, backticks, dash flavour and wording are free.                    |
| Partial credit                     | Score is the weighted share of passed checks. Severity and output-format checks add credit but never gate. Only required checks decide pass/fail. |
| Unambiguous, solvable tasks        | One seeded defect per task. `references/<id>.md` is a known-good review; `grader.spec.ts` proves the grader gives it a perfect score.            |
| Grader transparency                | Every check is derivable from `tasks.ts` and the skill's own output contract. The prompt names the goal, not the steps.                           |
| Balanced positive/negative         | 5 tasks must produce a finding; `clean-change` and `style-only-change` must not (false positives and out-of-scope lint/format noise).            |
| Isolation                          | Each trial builds a fresh temp git repo (`main` + `feature`) from `fixtures/`. No shared state; deleted afterwards.                               |
| Non-determinism                    | `EVAL_TRIALS` runs N trials per task. `eval-reporter.ts` prints pass@k (any pass) and pass^k (all pass).                                          |
| Environment state check            | `outcome:workspace-unchanged` and `outcome:no-commits` verify the skill stayed report-only, with edits left enabled so a violation is visible.   |

## Layout

- `grader.ts` — parser + `gradeReview(report, task, outcome?)`; pure, no I/O
- `tasks.ts` — task definitions (expected findings, forbidden findings, severity caps)
- `fixtures/_base/` + `fixtures/<task-id>/` — base repo and per-task overlay; files carry a `.txt`
  suffix so tsc/eslint ignore their intentional defects (stripped on copy)
- `references/` — known-good reviews, one per task
- `workspace.ts` — sandbox builder, outcome inspection, headless `claude` runner
- `grader.spec.ts` — self-tests (reference passes, wrong reviews fail, wording variation tolerated)
- `agent.eval.ts`, `eval-reporter.ts`, `playwright.config.ts` — live run and aggregation

## Adding a task

1. Add `fixtures/<id>/` with only the files the change touches (mirror the real repo paths, add `.txt`).
2. Seed **one** defect. Add an entry to `tasks.ts` with the file, line range, `§` and keywords.
3. Write `references/<id>.md` in the skill's output format. `npm run eval:review:grader` must pass.
4. Keep the set balanced: for every "must flag" task, consider a nearby "must not flag" one.

## Known limits

- Keyword and line-range matching is deterministic but can miss a valid finding phrased in an
  unexpected way. When a live trial fails on a correct review, widen the task's `keywords`.
- Test execution is skipped in the sandbox (no browsers/network), so the skill's step 5 is not
  covered; `tests:` is only checked for presence.
- Judging fix quality or tone would need a model-based grader; this one is code-only by design.
