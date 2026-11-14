# Loop control (Step 2)

How the orchestrator runs the review-fix loop so it ends. The reviewer's severity rules are in
the plugin's [agents/pr-reviewer.md](../../../agents/pr-reviewer.md).

Reviews are cheap; fix rounds are not. On eggd_cgp-cnvkit-batch (2026-11-04) the reviews were
about 4% of the cost and the fix agents 65%. Each full review sampled a fresh set of findings
rather than converging, and later rounds mostly chased unreachable variants of one defect class.
So the loop is **one wide review, one triage, narrow checks of the fixes, one final round**.

## Setup (once per task, before the wide review)

`<meta>` below is the main checkout's git directory as an absolute path
(`git rev-parse --path-format=absolute --git-common-dir`). It works from linked worktrees too, where
`.git` is a file. Each task gets its own directory, **`<task>` = `<meta>/pr-review/branches/<branch-name>/`**
(create it with `mkdir -p`, keeping any `/` in the branch name as a subdirectory — git forbids branches `a` and `a/b` existing together, so paths can't collide; on a detached HEAD, e.g. the review-only path, use `<meta>/pr-review/detached/pr-<number>/` — a separate namespace, so no branch name can map onto it), so concurrent tasks in linked
worktrees never share a ledger, PR body or prompt file. Every step — Step 2b's `gh pr create` /
`gh pr edit` included — reads and writes only that task's files. Pass them to the reviewer by
absolute path.

- **Clean worktree.** Every `pr-reviewer` call runs with `isolation: "worktree"`, so another
  session's untracked files never block HEAD certification. Don't create review worktrees
  yourself: the sandbox makes `/tmp` and `.git/worktrees` read-only, so `git worktree add` fails.
  Claude Code builds the isolated worktree from the default branch, not your HEAD, so every
  prompt names the HEAD SHA and the reviewer checks it out first (worktrees share the object
  store, so unpushed local commits are available). Commit before every round — the reviewer
  sees committed work only.
- **Ledger** in `<task>/ledger.md` (untracked). It belongs to the **task**, not the loop:
  keep it across resets, narrowing and restarts, with each finding's actual status (a finding
  `fixed in <sha>` on a discarded commit goes back to `open`; a fixed one still needs its
  verification). One line per finding:
  `F<n> | class | round | severity | original text | reachable? | status`, where status is one of
  `open`, `fixed in <sha>`, `explained (new|accepted in round N): <argument>`,
  `decided by user: <decision>`, `deferred to <link>`, `🔵 not applied`. `class` is a short name
  for the defect class (e.g. "PAR row↔segment attribution under overlapping segments"). Keep a
  per-round row: `round | type (wide/verify/final) | 🟡 count | reachable in-purpose 🟡s`, and a
  running count of full rounds.
- **Deployment facts** in the ledger: the real inputs and configurations this code meets (panels,
  file properties, who runs it), each with how it was checked (a command and its output).
- **Draft PR body** in `<task>/pr-body.md` — for an existing PR, start from its current body
  (`gh pr view <n> --json body --jq .body` as a bare call, with no `>` redirect, which would keep
  `gh` sandboxed and failing; then write the output to the file with the Write tool), and save that fetched body as `<task>/pr-body.base.md` the same way: summary, deferrals, 🔵s not applied, known
  limitations, and test-run evidence. The reviewer checks its claims; Step 2b publishes it.
- **Prompt file.** Write each review prompt to `<task>/prompt-<round>.md` (Write tool) in
  a tool call *before* the one that launches the reviewer, and check it with `test -s`. Pass its
  full text as the Agent prompt — the isolated reviewer can read `<task>` by absolute path, but
  don't rely on it to go and find the file.
- **Skill freshness.** At the start of each round, if `SKILL.md` or a file in `references/` has
  changed since you last read it, read it again.
- `<meta>` is inside the repo's `.git`, which the sandbox allows writing to (checked 2026-11-13).
  `gh` runs outside the sandbox and reads `<task>/pr-body.md` by absolute path.

## What goes into every prompt

- Base commit, HEAD SHA, purpose, plan file (if any), the draft PR body as claims to check.
- **Usage context**, one line per part of the change: production/unattended, CI or test harness,
  or hand-run docs/scripts. Plus the **deployment facts** and the **release/process context**
  (who builds, signs off and publishes).
- From the ledger: `Decided by user`, `Explained` (with the argument), `Deferred to <link>`.
- Static-analysis output (references/static-analysis.md).

## 1. Wide review (once per task)

Run 2–3 `pr-reviewer` calls in parallel on the first SHA, each with the full prompt. For a large PR
(more than ~20 files or ~1,500 changed lines), these are the shards (app code, tests/harnesses,
docs, CI/infra) instead, each prompt listing its files: together they must cover every changed,
deleted and renamed path (put anything else in the nearest shard), all at the same base and SHA. Merge the findings, drop duplicates, and give each an ID and a class.

## 2. Triage with the user (once, in one table)

For each 🟡, first check **reachability**: can its trigger occur in the deployment facts? Run the
check (e.g. count overlapping intervals in the real BED) and record the result. Then give the user
one table:
`ID | class | finding | realistic trigger | reachable? (evidence) | proposed: fix / explain / defer`.
- An unreachable trigger defaults to explain or defer; fixing it needs the user's yes.
- **Policy questions, asked here once:**
  - For guards and self-checks: "Is a safe failure (the job stops, nothing wrong is published) on
    rare valid input acceptable?"
  - For any compatibility rule: what scope does it cover?
  - **After-triage policy:** where do non-blocking findings found later go (default: the PR body
    as known limitations, plus a named tracking issue)? See §4.
  
  Never record an unscoped universal rule ("every valid input stays identical") as `Decided by
  user`. Scope it to the deployment facts.
- Record the answers. Create agreed tracking issues straight away (allowed during the loop).

## 3. Fix, then verify narrowly

- **Fixes.** The smallest change that removes the realistic risk. Never apply 🔵s. If fixes would
  add more code than the PR's original change, stop and ask. Fixes of about 20 lines or fewer,
  and docs fixes, are made by the orchestrator directly, not a new fix agent.
- **Tests.** Fix agents run targeted tests only. The orchestrator runs slow suites (platform or
  validation runs, and slow repo CI checks) at most once per HEAD, and only if
  `git diff --name-only <last-run-sha> HEAD` touches files they depend on; otherwise it passes the
  recorded result from `<last-run-sha>` to the reviewer, with that diff as the reason it still holds. Record the evidence in `<task>/pr-body.md`, not in commits.
- **Verification round.** A `pr-reviewer` call whose prompt names a **verification scope**: the
  fix commits (`git log --oneline <before>..HEAD`), and for each, the finding ID, class and
  original text. The reviewer judges each fix and looks for new problems the fix introduced,
  within those commits. An explanation, or a wording change with no behavioural effect, gets a
  judge-only prompt: the original finding and the new wording or explanation, asking "Does this resolve the finding's risk, and is it
  correct?" with the context needed to judge (e.g. the older releases a rollout note must cover).
  Changes to skill, agent or other instruction files are behaviour, not wording: they get the full
  fix-commit verification above (including new problems introduced), never judge-only.
- Repeat fix → verify until every agreed 🟡 is fixed, explained (and accepted), or deferred.

## Stop rules (check after every Not-clean round before the final)

A Clean verification round before the final goes on to the final round; after the final, §4
governs. After a Not-clean round, **pause and ask the user**, with the triage table for what is open, if any of these hold:
- **Class repeat.** A 🟡 lands in a class that already had one. Ask for one class-level decision,
  scoped to its trigger and consequence (e.g. "overlapping segments: a safe job failure is
  accepted"). Later findings in the class with the same consequence are covered; one with a worse
  consequence (e.g. a wrong result is published) is a new question for the user.
- **No gain.** Two consecutive rounds produced no reachable, in-purpose 🟡.
- **Return.** A finding comes back a second time after being fixed: by default it is explained or
  deferred, not strengthened again — unless the user, at this pause, decides to try another fix
  (within the verification budget).
- **Budget.** Full rounds for this task have reached the cap.

## 4. Final unscoped round (one per task)

Only once the agreed 🟡s are all closed (and any 🔵s the user agreed to are applied in one commit).
It is a full prompt with no focus areas and no verification scope, with the ledger's decisions,
explanations and deferrals; shard it for a large PR, covering every changed path as in 1.

The final **confirms**; it does not restart discovery. Each shard reports its own verdict. Every
new 🔴 blocks the push. A new 🟡 **blocks the push** only if all of these hold:
- it is within the PR's purpose (the agreed scope in `Decided by user`);
- it is reachable in the deployment facts;
- its usage context is production/unattended code, or it would make a published result or a
  release decision wrong;
- it is not in a class already deferred to an issue (add it to that issue instead).

Exception: a 🔴/🟡 that is a **material security exposure** — credentials, write tokens or secrets
reachable by untrusted code (e.g. a CI workflow that runs PR code with secrets or a write token) —
blocks the push whatever its usage context. A passing CI run does not rule it out.

**Certification.** Every final shard must certify the requested base and SHA (pr-reviewer.md →
Constraints), and so must any post-final verification whose SHA would become the pushable SHA. A
verdict that does not certify HEAD is never a non-blocking finding: fix the cause and rerun the
round (a process failure, so it doesn't count against the budget) — at most two reruns; if the
second rerun is also uncertified, stop and tell the user.

Every other new 🟡, and every 🔵, goes where the after-triage policy says (default: the PR body as
a known limitation, and the tracking issue), with no pause for the user. Then:
- **Nothing blocking:** the final's SHA is the **pushable SHA**; go to Step 2b. The PR body lists
  what was recorded.
- **Blocking findings:** triage them with the user (as in 2), fix, and verify the fixes narrowly (as
  in 3). The SHA of the Clean verification that closes the last of them is the pushable SHA; go to
  Step 2b. There is no second full final round.

After the final, this section governs: the stop rules and the full-round cap do not pause for
non-blocking findings, and a Clean post-final verification goes to Step 2b, not to another final.

**Later feedback (Step 3).** Fixes for review comments after the push are the same task: triage,
fix, and verify them narrowly, using the remaining verification rounds; they need no new wide or
final round — **but only if `<task>/ledger.md` records a completed wide review and final for this
PR.** If it doesn't (the PR predates this workflow, or the ledger is missing), run Setup and the
full loop from step 1 instead. Every publish of `<task>/pr-body.md` follows the PR-body guardrail
in SKILL.md (re-fetch, compare with `<task>/pr-body.base.md`, merge), so GitHub-side edits are
never overwritten. The Clean verification's SHA is the pushable SHA. Feedback that changes the PR's
purpose or adds new work is a new task, with its own ledger section and budget.

## Budget

- **Full rounds per task: 2.** The wide review and the final. A round of parallel reviewers or
  shards counts as one. The count survives resets and narrowing.
- **Verification rounds: up to 5 per task.** Running out of verification rounds does not use up the
  final round: ask the user whether to run it with the open items deferred or listed.
- A round that failed for a process reason (the reviewer did not get its prompt, a tool error)
  does not count; fix the cause and rerun it (uncertified-verdict reruns are capped at two, see §4).
- **Budget spent** (verification rounds used up with a blocking 🟡 still open): stop, do not push. Offer the user: ship with the ledger's open items listed in
  the PR body as known limitations, defer them, or extend the budget.
