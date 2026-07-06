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
| "review this PR" / "/pr-workflow review" | Step 2 — Review-fix loop (review pass only) |

Never proceed from one step to the next without explicit instruction from the user.

---

## Step 1 — Branch & commit

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

Run up to 10 rounds.

**Each round:**

1. **Fix subagent** — given the current findings (first round: none, so it reviews the local state from scratch); triages; applies fixes locally; commits (no push). Reports: what was fixed / explained / skipped, and the commit hash.

2. **Review subagent — always the bundled `pr-reviewer` agent from this plugin, never `general-purpose`.** It's a blank-slate agent (`agents/pr-reviewer.md` in this plugin — single source of truth for the review framework, don't restate it here) with the model already configured — don't pass `model` in the call. Reports: findings table + clean/not-clean verdict.

   **Pass settled facts.** The reviewer has no memory of previous rounds, so without a `Known facts (do not re-investigate)` block listing what's already verified or deliberately decided — e.g. `"Django test runner creates data/ if absent — CI proves this"` — it re-raises the same triaged findings every round.

3. If clean → exit loop and proceed to Step 2b. Otherwise repeat.

**Step 2b — Push and open/update PR:**
```bash
git push -u origin <branch-name>          # first time
git push                                   # subsequent rounds
gh pr create --title "<70 chars>" --body "## Summary
- bullet

## Test plan
- [ ] check"
```
Print the PR URL. Stop — wait for explicit instruction before proceeding.

---

## Step 3 — Apply external feedback

Triggered when a reviewer has posted comments on the open PR. Fetch all comments, apply fixes locally, then **re-run the full review-fix loop (Step 2)** before pushing.

Fetch comments:
```bash
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments
gh api repos/{owner}/{repo}/issues/{pr_number}/comments
```

Triage each comment as **Fix**, **Explain**, or **Skip** (bot noise). Apply all fixes in one editing pass and commit locally (no push yet). Then run Step 2 loop. After the loop exits cleanly, push and reply to each actioned comment:

- Reply endpoint: `repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_id}/replies`
  ⚠️ Must include `{pr_number}` in path — omitting it returns HTTP 404.
- Use `--input -` with `json.dumps({"body": text})` — not `-f body=` (backticks and newlines break shell escaping).
- Fix reply: `"Fixed in \`{hash}\` — {one sentence}."`
- Explain reply: concise reasoning, no apology.

Single-user repos: same username appears as both author and reviewer. Treat 🔴/🟡/🔵-labelled comments as reviewer findings; treat `"Fixed in \`hash\`"` replies as already-actioned.

---

## Step 4 — Post consolidated review to GitHub (optional)

After loop exits and push is done, optionally post a single review:

```python
import json, subprocess
review = {
    "commit_id": "{head_sha}",           # gh pr view {n} --json headRefOid
    "body": "**Code Review — {title}**\n\n{summary}\n\n**What's done well:** {positives}.",
    "event": "COMMENT",                  # never REQUEST_CHANGES — blocked on own PRs
    "comments": [{
        "path": "{file}",
        "line": {line_number},           # file line number from diff @@ +N header
        "side": "RIGHT",                 # RIGHT = added/context; LEFT = removed
        "body": "🔴/🟡/🔵 **{title}**\n\n{explanation}\n\n```\n{fix}\n```"
    }]
}
subprocess.run(["gh", "api", f"repos/{owner}/{repo}/pulls/{pr_number}/reviews",
    "--method", "POST", "--input", "-"], input=json.dumps(review), capture_output=True, text=True)
```

Verify: `gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews/{review_id}/comments --jq '.[] | "\(.path) line \(.line): \(.body[:60])"'`

---

## Guardrails

- Never force-push or amend a pushed commit. Never use `--no-verify`.
- Do not mark a PR ready-to-merge unless explicitly asked.
- Loop: no `git push`, no GitHub API writes until the loop exits cleanly.
- Inline comments: always `line` + `side` — never the legacy `position` field.
- Review payload: always `--input -` via stdin — nested JSON arrays break with `--field`.
