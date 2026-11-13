---
name: pr-workflow
description: Commit and raise a pull request; run the local review-fix loop; respond to PR review comments; review a PR and post inline comments. Triggers on "commit and raise a PR", "open a pull request", "respond to review comments", "address PR feedback", "/pr-workflow review".
allowed-tools: Bash, Read, Glob, Grep
---

## GitHub Flow

```
branch → commit → [loop] → push → open PR → [loop] → push → [loop] → push → … → merge
                  ↑ local                    ↑ local           ↑ local
                  only                       only              only
```

The pattern repeats: **changes locally → run loop until clean → push once** — for the initial commit and for every round of external reviewer feedback (see Guardrails for the push gate).

Entry points:

| Trigger | Start at |
|---|---|
| "commit and raise a PR" | Step 1 — Branch & commit |
| "respond to review comments" / "address PR feedback" | Step 3 — Apply external feedback |
| "review this PR" / "/pr-workflow review" | Step 2, review only: get the PR's SHAs (`gh pr view <n> --json baseRefName,headRefOid,baseRefOid`), fetch them (`git fetch --prune origin pull/<n>/head <base>`), and review `headRefOid` against `git merge-base <baseRefOid> <headRefOid>` as the base. Run Step 2 steps 1–2 (static analysis, the wide review, given the actual PR body as claims to check) once — not Step 3, and no replies — report the findings, stop. No fix subagent, no commits, no push, no PR edits (report a missing Jira link instead); Step 4 only if asked |

Never proceed from one step to the next without explicit instruction from the user. Exception: a single trigger like "commit and raise a PR" authorizes the whole Step 1 → 2 → 2b chain in one go. Inside that chain, Step 2's triage with the user (and any stop-rule pause) is the planned stop: once the user has answered, carry on to Step 2b without asking again. The "no proceeding without instruction" rule is about not starting a *new* step (e.g. Step 3, or a fresh commit) on your own initiative after a chain completes.

Every PR body links its Jira issue — see the Jira link guardrail below.

---

## Step 1 — Branch & commit

Local git (`status`, `diff`, `checkout`, `add`, `commit`, `fetch`) works fine inside the sandbox and can be chained. Only `gh` and `git push` need single bare calls — see the sandbox guardrail.

```bash
git status && git diff          # understand what's changed
git checkout -b <kebab-name>    # descriptive branch name
git add <specific files>        # never git add -A or git add .
git commit -m "<type>: <imperative summary>

<optional: why, not what>"
```

Commit types: `feat` `fix` `docs` `refactor` `test` `chore`. Never add `Co-Authored-By`.

Proceed immediately to Step 2.

---

## Step 2 — Review-fix loop (local only)

The loop is **one wide review, one triage with the user, narrow checks of the fixes, one final round** — see [references/loop-control.md](references/loop-control.md) for the setup (task ledger with defect classes and deployment facts, draft PR body, prompt files), the stop rules and the budget (2 full rounds per task: the wide review and the final). Every review prompt names the **base** and the HEAD SHA. `git fetch --prune` first, and give the base as a commit (`origin/<base>`), never a possibly stale local branch. For an existing PR, before any fixes (Step 1 or Step 3), check the local branch is at the PR's `headRefOid` (`gh pr view <number> --json baseRefName,baseRefOid,headRefOid`), record those, and take the base from it; just before pushing, query them again and check the base name, base OID and remote head are what you recorded. If any changed, rerun the loop.

**Launching the reviewer.** Always the bundled agent — `subagent_type: "eastgen-plugin:pr-reviewer"`, never `general-purpose` — with **`isolation: "worktree"`**. It's a blank-slate agent ([agents/pr-reviewer.md](../../agents/pr-reviewer.md) — single source of truth for the review framework, don't restate it here) with the model already configured — don't pass `model` in the call. The isolated worktree is created from the default branch, so the prompt must name the HEAD SHA: the reviewer checks it out (`git checkout --detach <sha>`) before reviewing. Only **committed** work is visible to it — commit before every review round. Reports: findings table, clean/not-clean verdict, and the HEAD SHA it reviewed.

**Steps:**

1. **Static analysis and sweeps** — run [references/static-analysis.md](references/static-analysis.md) on the changed files and paste the output into every review prompt.

2. **Wide review** — 2–3 `pr-reviewer` calls in parallel (one message, several Agent calls) on the first SHA (shards for a large PR). Merge, dedupe, and give each finding an ID and a defect class.

   **What goes into the prompt.** The reviewer has no memory of previous rounds, so pass a `Known facts` block, or it re-raises the same triaged findings every round — but only for **decisions and environment facts** (e.g. `"Django test runner creates data/ if absent — CI proves this"`, `"X is deliberate because Y"`).
   - Never list fix code as settled: write "Fixed since round N — verify the fix itself", and give the **original finding text**, not a description of what the fix does.
   - Never give tool verdicts ("bandit clean", "tests pass") as a reason not to look. For outside facts the reviewer cannot check (API behaviour, docs, GitHub issues), paste the source excerpt or command output, not "verified".
   - Outside a verification round, a "focus on X" note is a priority, never the scope: always add "and anything else in the full diff". Never write "pre-existing, so low severity" or "file Y is out of scope".
   - List the changed docs (README, CHANGELOG, specs), and give the commit messages and the draft PR body (`<meta>/pr-body.md`, see loop-control.md) as claims to check. If the work follows a plan file, give its path and say which parts are out of scope.
   - Give the **usage context** of each part (production/unattended, CI or test harness, hand-run docs/scripts), the **deployment facts** with their evidence, and the release/process context (who builds, signs off, publishes), and the ledger's `Decided by user`, `Explained` and `Deferred to` lists.

3. **Triage with the user, once** — check each 🟡's reachability against the deployment facts, and ask the policy questions (loop-control.md §2). One table: fix / explain / defer. Unreachable triggers default to explain or defer.

4. **Fix, then verify narrowly** — smallest fixes (the orchestrator makes small and docs fixes itself; never 🔵s), targeted tests, then a verification round scoped to the fix commits and their finding IDs (judge-only for docs or explanations). Check the stop rules after every round: a repeat of a defect class, two rounds with no reachable in-purpose 🟡, a finding returning after a fix, or the budget — any of them means pause and ask the user. A valid finding outside the PR's purpose goes to a tracking issue (with the user's agreement) and is listed as `Deferred to <link>: <original finding text>`.

5. **Final unscoped round** — once the agreed 🟡s are closed (and agreed 🔵s applied in one commit): a fresh full `pr-reviewer` call, with step 1 re-run at that SHA, no focus areas and no verification scope, but with the ledger's decisions, explanations and deferrals (shard it for a large PR; every shard must report the same base and SHA). It confirms rather than discovers: a new 🟡 blocks the push only if it is in the PR's purpose, reachable, and in production code (or would make a published result or release decision wrong), and not in an already-deferred class; everything else goes to the PR body and tracking issue under the after-triage policy, without pausing (loop-control.md §4). Blocking 🟡s are triaged, fixed and verified narrowly — there is no second full final. If the budget is spent, stop — do not push — and offer: ship with known limitations in the PR body, defer, or extend.

**Step 2b — Push and open/update PR:**

Resolve the Jira link and check the push gate (both in Guardrails) before pushing. `<meta>/pr-body.md` (see loop-control.md → Setup) is the PR body: `## Summary`, deferrals, 🔵s not applied, test evidence, then

```
Local review: <N> rounds; final at <sha> (<k> non-blocking findings recorded); pushable SHA <sha>
Jira: <issue-url>
```

Then run each of these as its **own** Bash call, from the repo's working directory (see the sandbox guardrail). Every round pushes only the reviewed SHA:

```bash
git push origin <reviewed-sha>:refs/heads/<branch-name>
```

New PR only:

```bash
git branch --set-upstream-to=origin/<branch-name>
```
```bash
gh pr create --base <base> --head <branch-name> --title "<70 chars>" --body-file <meta>/pr-body.md
```

Existing PR — no `gh pr create`; publish the reviewed draft (it started from the current body):

```bash
gh pr edit <number> --body-file <meta>/pr-body.md
```

Then check `gh pr view <number> --json baseRefName,headRefOid` matches the reviewed base and SHA. Print the PR URL. If you came here from Step 3, go back and finish its replies, then stop. Otherwise stop — wait for explicit instruction before proceeding.

---

## Step 3 — Apply external feedback

Before anything else, check the PR carries its Jira link (see Guardrails). Then fetch all comments, apply fixes locally, then **re-run the review-fix loop (Step 2)** before pushing.

Fetch **all three**, each as its own bare Bash call, and read every body in full (no truncation, no filtering by author). `gh` fills in `{owner}`/`{repo}`; fill in `{pr_number}` yourself. If a fetch fails, stop.

```bash
gh api --paginate repos/{owner}/{repo}/pulls/{pr_number}/comments
```
```bash
gh api --paginate repos/{owner}/{repo}/issues/{pr_number}/comments
```
```bash
gh api --paginate repos/{owner}/{repo}/pulls/{pr_number}/reviews
```

CodeRabbit puts its "Nitpick comments", "Outside diff range comments" and "Duplicate comments" only in review bodies — triage every item there too.

Triage each item as **Fix**, **Explain**, or **Skip**, with a reason. Apply all fixes in one editing pass and commit locally (no push yet). Then run Step 2 from step 3: triage with the user, fix, and verify narrowly with each original comment text as the finding (the task's ledger and verification budget carry over; no new wide or final round — loop-control.md §4, Later feedback). After the loop exits cleanly, push (Step 2b) and reply to every item, bot nitpicks included. If nothing is a Fix, skip the commit, loop and push, and just post the replies (plus any agreed deferral-ledger line in the PR body), without a `Local review:` line:

- Never decline an item only by adding it to Known facts — it gets a reply.
- Fix the risk, not the wording: if a comment pairs a mismatch with a risk, changing the docs to match the code is not a fix.
- Before replying "Fixed", `git grep` for the same pattern across the repo and re-check every location the comment names. CodeRabbit's "✅ Addressed" is not proof.
- If the code came from a template or a sibling repo, name the sibling repos with the same code in the reply, and ask the user whether to raise follow-ups.
- Deferral ledger: every valid-but-deferred item, including "handled in stacked PR #N", gets a `Deferred to <link>: <original finding text>` line in the PR body (`<link>`: a tracking issue, or the stacked PR), carried into every later review prompt, and checked when that PR merges. In a PR stack, fix an issue in the PR that introduced it, not only downstream.
- Re-fetch all three endpoints after every push, and again before reporting the PR as ready to merge. Triage anything new the same way (fixes go through the Step 2 loop); never report ready while an item is unanswered.

Reply mechanics:

- Each reply batch after a push states `Local review: <N> rounds; pushable SHA <sha>, verified narrowly` once (in the first reply or in the issue comment below).
- Review-body items (nitpicks, outside-diff) and issue comments have no reply endpoint: answer them in one issue comment (`repos/{owner}/{repo}/issues/{pr_number}/comments`), one line per item, quoting or linking the item it answers.
- Reply endpoint: `repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_id}/replies`
  ⚠️ Must include `{pr_number}` in path — omitting it returns HTTP 404.
- Write `{"body": "<text>"}` to a JSON file (Write tool, or `json.dump` in a separate Python call), then `gh api repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_id}/replies --method POST --input <scratchpad>/reply.json` as a bare call — not `-f body=` (backticks and newlines break shell escaping), and not `--input -` from a pipe or heredoc (that keeps the call sandboxed). Same pattern for the issue comment. Throughout this skill, `<scratchpad>` means the session's scratchpad directory from the environment context (an absolute path), or `$TMPDIR` if there is none.
- Fix reply: `"Fixed in \`{hash}\` — {one sentence}."`
- Explain reply: concise reasoning, no apology.

Single-user repos: same username appears as both author and reviewer. Treat 🔴/🟡/🔵-labelled comments as reviewer findings; treat this workflow's own replies (`"Fixed in \`hash\`"`, Explain/Skip, the review-body issue comment) as answers, not new items.

---

## Step 4 — Post consolidated review to GitHub (optional)

After the loop exits and push is done — or, on the review-only path, when the user asks — post a single review with the findings (Clean or not), against the SHA that was reviewed (`commit_id`); if the PR head has moved since, say so and ask before posting.

Build the payload in Python and write it to a file — **don't** call `gh` from Python: a `python …` call doesn't match the sandbox's `gh *` exclusion, so `gh` would run sandboxed and fail authentication.

```python
import json
review = {
    "commit_id": "{reviewed_sha}",       # the SHA pr-reviewer reported, not the current head
    "body": "**Code Review — {title}**\n\n{summary}\n\n**What's done well:** {positives}.",
    "event": "COMMENT",                  # never REQUEST_CHANGES — blocked on own PRs
    "comments": [{
        "path": "{file}",
        "line": {line_number},           # file line number from diff @@ +N header
        "side": "RIGHT",                 # RIGHT = added/context; LEFT = removed
        "body": "🔴/🟡/🔵 **{title}**\n\n{explanation}\n\n```\n{fix}\n```"
    }]
}
with open("<scratchpad>/review.json", "w") as f:
    json.dump(review, f)
```

Then, as its own Bash call:

```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews --method POST --input <scratchpad>/review.json
```

Verify: `gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews/{review_id}/comments --jq '.[] | "\(.path) line \(.line): \(.body[:60])"'`

---

## Guardrails

- Never force-push or amend a pushed commit. Never use `--no-verify`.
- Do not mark a PR ready-to-merge unless explicitly asked.
- Loop: no `git push`, no GitHub API writes until the loop exits cleanly — except creating a tracking issue the user agreed to, the Jira-link PR body edit, Explain/Skip-only replies and their deferral-ledger body line (Step 3), and a review the user asked to post (Step 4).
- **Push gate — push only what was reviewed.** Before any `git push` or `gh pr create`, `git rev-parse HEAD` must equal the **pushable SHA** (loop-control.md §4): the final round's SHA when it found nothing blocking, or the SHA of the Clean narrow verification that closed its blocking findings (or a Step 3 fix). Any later commit — applying 🔵s, a CHANGELOG tweak, a Step 3 fix — reopens the loop. This applies however the push comes about (another skill, another task, a subagent).
- A push that changes `.github/workflows/` over HTTPS with the `gh` credential needs the `workflow` scope (classic OAuth token). If it is rejected for that, stop and ask the user to run `! gh auth refresh -h github.com -s workflow` themselves (it opens a browser).
- If `pr-reviewer` cannot run (auth error, outage, no Bash tool, or it cannot check out the SHA), stop and tell the user. Never offer "push anyway", never substitute a `general-purpose` reviewer.
- Docs, rollout and infrastructure-advice checks belong to the reviewer's `Docs & claims` dimension. The skill's job is to give it the sweeps, the plan path and the changed-docs list.
- Inline comments: always `line` + `side` — never the legacy `position` field.
- Review payload: always `--input <file>` — nested JSON arrays break with `--field`.
- **Sandbox: run `gh` and `git push` as single, bare Bash calls.** The team's managed settings exclude only `gh *` and `git push *` from the sandbox, and an exclusion applies only when *every* part of the call matches. A `cd` first, chaining (`&&`, `;`, `|`), a redirect, a heredoc or pipe into stdin, `$(...)`, or calling `gh` from Python/a script all keep the call sandboxed — where `gh` can't reach the OS keyring, so it fails with "token … is invalid" / HTTP 401 and `git push` fails with "could not read Username". Run from the repo's working directory instead of `cd`-ing, pass bodies/payloads via `--body-file <file>` / `--input <file>`, and don't conclude that `gh`/`git push` can't work from the agent until a bare call has failed. (`git -C <dir> push` doesn't match `git push *` either.) `git fetch` is not excluded: it works sandboxed for public repos; if it fails with "could not read Username" on a private repo, ask the user to run it with `! git fetch …`.
- **Jira link, single source of truth for the whole skill** — every PR body carries a `Jira: <issue-url>` line. Resolve the issue key from the branch name if it follows `<KEY>-<n>-description` (e.g. `abc-123-fix-thing` → `ABC-123`); otherwise ask the user. Look it up via the bundled `atlassian` MCP server for the canonical URL — never guess the site domain. Check for it yourself in the main session — the `pr-reviewer` subagent is disk-only and never sees the PR body: `gh pr view <number> --json body --jq .body`. Check before `gh pr create` (Step 2b) and again as the first action of any "review this PR"/Step 3 entry, before the subagent loop starts; if missing, write the corrected body to a file and run `gh pr edit <number> --body-file <file>` to add it before proceeding — except on the review-only path, which only reports it.
