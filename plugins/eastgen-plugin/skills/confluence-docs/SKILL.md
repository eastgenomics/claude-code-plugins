---
name: confluence-docs
description: "create dev-doc | create resource-doc | create policy-doc | update — create and update controlled documentation pages in the CUH Bioinformatics Documentation Vault on Confluence."
argument-hint: "create dev-doc | create resource-doc | create policy-doc | update"
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

You are helping manage controlled documentation in the CUH Bioinformatics Documentation Vault on Confluence.

The user has invoked this skill with the following arguments:

$ARGUMENTS

---

## Mode dispatch

Read the arguments, determine the mode, then **immediately read the corresponding reference file** before doing anything else:

| Arguments start with | Reference file | Covers |
|---|---|---|
| `create dev-doc` | [references/dev-doc.md](references/dev-doc.md) | Development & Testing docs — apps, tools, config files |
| `create resource-doc` | [references/resource-doc.md](references/resource-doc.md) | Controlled File docs — data files, variant stores, reference resources |
| `create policy-doc` | [references/policy-doc.md](references/policy-doc.md) | Manual/Policy docs — operational guidance, training policies |
| `update` | [references/update.md](references/update.md) | Surgical edits to any existing page |

Text after the mode keyword is additional context (title, URLs, description of the change) — carry it through into the reference file's workflow. If no valid mode is given, ask which of the four is needed.

Read [../../shared/confluence/HTML_DIALECT.md](../../shared/confluence/HTML_DIALECT.md) once per session before writing any page body — every mode below builds HTML+ content (panels, status lozenges, expand sections, Mermaid), and that reference is the single source of truth for the exact shapes.

---

## Org constants

- **Base URL**: `https://cuhbioinformatics.atlassian.net/wiki`
- **Space key**: `DV`
- **Space ID**: `2903080965`
- **Cloud ID**: `3419b8d5-6218-492f-bff8-812d5d24cdc7`

Confirm these are still current with `getConfluenceSpaces` / `getAccessibleAtlassianResources` if anything looks off — they're recorded here to skip a lookup on every run, not treated as unchangeable.

## Access

Use the bundled `atlassian` MCP server (`ToolSearch: query "confluence"`) — OAuth-authenticated per user, no token file. If a call reports the server needs authentication, direct the user to `/mcp` or `claude mcp login atlassian`.

There is no MCP tool for fetching a Confluence page **template** — the three `create` modes below build each page's full HTML+ body directly from that reference file's Content Guide instead of starting from a live template. The historical template IDs are noted in each reference file for provenance; they're not used at runtime.

---

## Shared conventions across all modes

**Resolving a parent page ID from a URL:** `https://cuhbioinformatics.atlassian.net/wiki/spaces/DV/pages/1234567890/...` → `1234567890`.

**Status lozenges** (see HTML_DIALECT.md): DRAFT = yellow, APPROVED = green, FAIL = red.

**Heading emojis:** existing pages in this vault prefix each top-level heading with a unique animal emoji (🐘🦁🐬🦅🐝🦊🦉🐙🦒🐺🐢🦋🐠🦈🐸🦩🐻🦚🐳🦜 …). Match this convention on new pages and preserve it on edits — skip a heading that already has one.

**Done when** (create modes): the page exists in Confluence with every Content Guide section populated (or explicitly marked "To be completed" where the reference allows it), and its URL has been reported back.

**Done when** (update mode): the specific requested change is applied, verified by re-reading the page, and any remaining manual step (permissions, archiving, ticket updates) from that reference file's Post-Update Reminders has been surfaced to the user.
