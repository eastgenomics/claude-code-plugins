# claude-code-plugins

Claude Code plugin marketplace for East Genomics.

## Plugins

- **eastgen-plugin** — bundles shared skills and agents:
  - `specification` — generate a full design/build spec for a software project
  - `pr-workflow` — commit/raise-PR, local review-fix loop, respond to PR feedback, post inline review comments
  - `pr-reviewer` (agent) — blank-slate, read-only five-dimension code reviewer used by `pr-workflow`
  - `confluence-code` — publish an annotated code walkthrough as a linked set of Confluence pages
  - `confluence-docs` — create/update controlled documentation pages in the CUH Bioinformatics Documentation Vault
  - `dnanexus` — build/deploy DNAnexus apps and applets, manage data with `dx`/dxpy, run and monitor jobs, navigate the Jira/GitHub/Confluence release process
  - `shared/confluence/HTML_DIALECT.md` — the Confluence HTML+ component reference both confluence skills use
  - `mcpServers.atlassian` — the official Atlassian remote MCP server (Confluence + Jira Cloud), OAuth-authenticated per user
  - `mcpServers.dnanexus-documentation` — GitBook's MCP server for the official DNAnexus documentation site (search, page fetch, sourced Q&A, doc-issue feedback); read access is unauthenticated (verified), the feedback-submission path wasn't separately checked. Used by the `dnanexus` skill

## Team setup

Note: the marketplace is named `eastgenomics`, not `claude-code-plugins` — that name is reserved for official Anthropic marketplaces (`github.com/anthropics/*` only), even though this repo is called `claude-code-plugins`.

Org admins should still add this to the Team's managed settings (Admin Settings > Claude Code > Managed settings), so the marketplace is registered and the plugin allowed org-wide. Paste the contents of [`team-managed-settings.json`](team-managed-settings.json) into that field — kept as a real file here (not just inline below) so it's copy-pasteable without transcribing it out of prose, and so it can't silently drift out of sync with a duplicate inline copy.

`autoUpdate` is opt-in and off by default for any marketplace you add yourself (`team-managed-settings.json` above already sets it `true`, so this only matters if you're registering the marketplace some other way, e.g. `/plugin marketplace add` below). Whichever way it's enabled, with `autoUpdate` on Claude Code checks for updates after session start (random delay up to ~10 min) and prompts `/reload-plugins` when one lands; without it, teammates only pick up new commits by running `/plugin marketplace update eastgenomics` manually. Either way, run `/plugin marketplace update eastgenomics` then `/reload-plugins` yourself for an immediate update instead of waiting.

**This paste is also an org-wide authorization, not just plugin registration:** it pre-authorizes `gh` CLI commands (skipping the interactive permission prompt) and unsandboxes both `gh *` and `git push *` for every Claude Code session org-wide, so both get real filesystem/network access instead of the sandbox's restrictions — needed because a sandboxed shell can't reach the OS keyring/credential helper `git`/`gh` use to authenticate, so `git push`, `gh pr create`, `gh api`, etc. otherwise fail with "could not read Username for `https://github.com`". `git push` still prompts for permission each time (unsandboxed but not pre-authorized); `gh` does not.

The `gh *` allow is deliberately broad (the intended use is PR/issue workflow — `gh pr`, `gh issue`, `gh api` against comments/reviews, which routinely means processing PR-comment text an external contributor wrote) but `gh` also exposes destructive operations and code-execution vectors unrelated to that, so `permissions.deny` carves out the ones identified so far — deny is checked before allow, so these stay blocked regardless of the broad `gh *` grant:
- Destructive data ops: `gh repo delete`/`archive`, `gh secret`, `gh workflow run`/`disable`/`delete`, `gh release delete`/`delete-asset`, `gh issue delete`, `gh auth logout`.
- Code-execution escape hatches — **the more important category, since this account processes untrusted PR-comment text**: `gh alias set --shell '<cmd>'` followed by `gh <name>` runs an arbitrary shell command; `gh extension install <repo>` (or its `gh ext`/`gh extensions` aliases) followed by `gh <extension-name>` runs downloaded third-party code; `gh config set editor/pager/browser <cmd>` can point `gh` at an attacker-chosen command it later shells out to. `gh alias`, `gh extension`/`gh ext`/`gh extensions`, and `gh config set` (not all of `gh config` — `get`/`list` stay allowed, since they're read-only) are denied outright.

**This deny list is curated, not exhaustive, and has known gaps beyond it we haven't closed:**
- It stops *creating* a new alias/extension via `gh` itself, but not *invoking* one that already exists (from before this policy, or written directly via a file-editing tool rather than `gh alias set`/`gh extension install` as a shell command) — same code-execution class, a different entry point we haven't addressed.
- `gh` subcommands that shell out to `git` under the hood (`gh pr checkout`, `gh repo sync`, `gh repo clone`) spawn `git` unsandboxed the same way `git push` does — a repo's `.git/hooks/*` or `.git/config` (`core.pager`, `core.sshCommand`) could be a comparable code-execution vector through `git` rather than `gh`, not evaluated here.
- **Known, accepted gap** (not just theoretical): `gh api -X DELETE`/`PATCH`/`POST` against an arbitrary endpoint is reachable, since `gh api` itself must stay allowed for reading/replying to PR comments — none of the destructive-op denies above can reach into a flag on an otherwise-allowed subcommand.
- **Genuinely open question, unverified**: whether an env-var-prefixed invocation (`GH_CONFIG_DIR=... gh ...`) still matches the `gh <subcommand>:*` deny patterns above, or bypasses them.

Each user still needs their own working `gh auth login` (or equivalent git credentials) set up locally — these settings remove the sandbox restriction, they don't provide credentials.

**Team setup also requires a per-teammate DNAnexus step for anyone using the `dnanexus` skill, not just the plugin install above:** each such teammate needs their own dedicated agent/service DNAnexus account (`MEMBER` org role, `CONTRIBUTE` project permission — never `ADMINISTER`) for Claude Code to authenticate as — never their own personal DNAnexus login. Account creation is per-person, documented in the [Onboarding for Claude](https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/4739137538/Onboarding+for+Claude) Confluence page; the token-wiring and verification steps (including why the account setup alone isn't sufficient — actual delete-safety depends on each DNAnexus *project*'s own `protected` flag, not the account) are in [`plugins/eastgen-plugin/skills/dnanexus/SKILL.md`](plugins/eastgen-plugin/skills/dnanexus/SKILL.md)'s `## Authentication` section. Skipping this means Claude ends up running DNAnexus commands with a person's full personal permissions.

**Version pinning gotcha — already fixed, but worth knowing about:** [`plugins/eastgen-plugin/.claude-plugin/plugin.json`](plugins/eastgen-plugin/.claude-plugin/plugin.json) had a static `"version": "0.1.0"` field for several commits. Setting `version` pins the plugin — Claude Code only delivers an update when that string changes, so every fix pushed while it was set was silently invisible to anyone who'd already installed the plugin, even with `autoUpdate: true`. The field is now omitted, so Claude Code tracks the underlying git commit instead and every push is a "new version." If you installed this plugin before this fix, run `/plugin marketplace update eastgenomics` then `/reload-plugins` once to unstick yourself from the frozen `0.1.0` cache — after that, ordinary updates (or `autoUpdate`) work as expected. (There is no separate `/plugin update <plugin>@<marketplace>` command — the marketplace is the unit of update, `/reload-plugins` is what applies a refreshed marketplace's changes to already-installed plugins.) Don't reintroduce a static `version` field in `plugin.json` without a process for bumping it on every release.

**Known limitation — one manual step per teammate on CLI/TUI:** managed-settings `enabledPlugins` auto-installs in the Desktop and web apps, but not in the Claude Code CLI. The original tracking issue, [anthropics/claude-code#45323](https://github.com/anthropics/claude-code/issues/45323), was closed as **not planned** — this isn't going to be fixed under that ticket. A fresh feature request is open at [anthropics/claude-code#91874](https://github.com/anthropics/claude-code/issues/91874); not something fixable from our side either way. After managed settings register the marketplace, each CLI/TUI user still needs to run this once:

```
/plugin install eastgen-plugin@eastgenomics
```

If a teammate reports "the marketplace shows up but the plugin isn't loaded," this is the expected cause — point them at the command above rather than re-diagnosing the managed-settings config.

Manual marketplace + install, without managed settings at all:

```
/plugin marketplace add eastgenomics/claude-code-plugins
/plugin install eastgen-plugin@eastgenomics
```
