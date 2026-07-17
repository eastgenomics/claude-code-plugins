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

## Team setup

Note: the marketplace is named `eastgenomics`, not `claude-code-plugins` — that name is reserved for official Anthropic marketplaces (`github.com/anthropics/*` only), even though this repo is called `claude-code-plugins`.

Org admins should still add this to the Team's managed settings (Admin Settings > Claude Code > Managed settings), so the marketplace is registered and the plugin allowed org-wide. Paste the contents of [`team-managed-settings.json`](team-managed-settings.json) into that field — kept as a real file here (not just inline below) so it's copy-pasteable without transcribing it out of prose, and so it can't silently drift out of sync with a duplicate inline copy.

`autoUpdate` is opt-in and off by default for any marketplace you add yourself — without it, teammates only pick up new commits by running `/plugin marketplace update eastgenomics` manually.

**Version pinning gotcha — already fixed, but worth knowing about:** `eastgen-plugin/plugin.json` had a static `"version": "0.1.0"` field for several commits. Setting `version` pins the plugin — Claude Code only delivers an update when that string changes, so every fix pushed while it was set was silently invisible to anyone who'd already installed the plugin, even with `autoUpdate: true` and a manual `/plugin update`. The field is now omitted, so Claude Code tracks the underlying git commit instead and every push is a "new version." If you installed this plugin before this fix, run `/plugin marketplace update eastgenomics` then `/plugin update eastgen-plugin@eastgenomics` once to unstick yourself from the frozen `0.1.0` cache — after that, ordinary updates (or `autoUpdate`) work as expected. Don't reintroduce a static `version` field in `plugin.json` without a process for bumping it on every release.

**Known limitation — one manual step per teammate on CLI/TUI:** managed-settings `enabledPlugins` auto-installs in the Desktop and web apps, but not in the Claude Code CLI ([anthropics/claude-code#45323](https://github.com/anthropics/claude-code/issues/45323), tracked upstream, not something fixable from our side). After managed settings register the marketplace, each CLI/TUI user still needs to run this once:

```
/plugin install eastgen-plugin@eastgenomics
```

If a teammate reports "the marketplace shows up but the plugin isn't loaded," this is the expected cause — point them at the command above rather than re-diagnosing the managed-settings config.

Manual marketplace + install, without managed settings at all:

```
/plugin marketplace add eastgenomics/claude-code-plugins
/plugin install eastgen-plugin@eastgenomics
```
