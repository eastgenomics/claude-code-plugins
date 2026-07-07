---
name: confluence-code
description: Publish an annotated code walkthrough for a source file as a linked set of Confluence pages, for a reader unfamiliar with the codebase and tech stack. Triggers on "document this code in confluence" or "/confluence-code".
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
---

# confluence-code skill

Produce a **linked set of Confluence pages** that explain a source file (or set of files) to a reader who is unfamiliar with both the codebase and the tech stack it uses — pages someone would actually read, not skim. This is the Confluence-native counterpart of a local HTML code-walkthrough: same four-part structure and content bar, different output medium and component library.

Read [assets/MACROS.md](assets/MACROS.md) (the Confluence storage-format component library) and [assets/PUBLISH.md](assets/PUBLISH.md) (how pages actually get created) before drafting or publishing anything.

---

## Before writing anything

1. **Read the source file(s) in full with the `Read` tool** before drafting a word.
2. **Identify the audience gaps.** What language concepts will be unfamiliar? What domain concepts? What external tools or libraries? These become Page 2's concept explanations.
3. **Confirm the Confluence target with the user** — space key, and (if this walkthrough belongs under an existing page) a parent page title or ID — whenever it isn't already established from context.
4. **Plan the page set** (see structure below). Estimate content per page. If Page 3 (function walkthrough) would cover more than ~15 functions or get unwieldy, split it into multiple sibling pages (`Function walkthrough — A`, `Function walkthrough — B`, …) rather than one huge page.
5. **Output the plan** before drafting any page — list every page's title and section assignments, and the page hierarchy (parent → children). Draft one page at a time and validate before proceeding.

---

## Page hierarchy

```
{Module/File} — Code Walkthrough      (index/parent page)
├── 1 · High-level outline
├── 2 · Key concepts
├── 3 · Function walkthrough          (split into 3a/3b/… if needed)
└── 4 · Line-by-line reference
```

The parent page is short: a one-paragraph orientation plus the Children Display macro (see MACROS.md).

Each child page opens with a Series navigation panel (see MACROS.md) so a reader who lands on it directly, without the space sidebar, can still navigate the set.

---

## Four-part structure

### Page 1 — High-level outline

Give the reader a complete orientation before they look at a single line of code.

- **What this file does** — one clear paragraph: what problem it solves, what it is not.
- **The N phases / stages** — a `status` lozenge per phase plus one sentence each, or a simple ordered list if a visual isn't warranted.
- **Data flow** — how data changes form as it moves through the file (e.g. raw file → parsed object → results list → output). A table.
- **External dependencies** — every import listed with what it is actually used for in this file. Separate standard library from third-party. Note anything imported lazily (inside a function) and why.
- **Function inventory** — a complete table: function name (inline code), line range, phase, one-line purpose.
- **Naming conventions** — any underscore prefixes, type hints, constant naming patterns.
- Insert the `toc` macro at the top of this page only (it's the entry point).

### Page 2 — Key concepts

Explain every concept a reader needs before the code makes sense. Assume the reader can write basic code in the language but may not know the specific libraries, patterns, or domain ideas used.

For each concept:
- **Plain-English explanation** from first principles.
- **Minimal code example** (`code` macro) showing the pattern in isolation.
- **"In this file"** — a `note` panel explaining exactly where and why this concept appears.

Candidates (pick those that actually apply — don't force irrelevant ones): language runtime behaviour, concurrency model, the specific framework/library idioms used, type system features, domain-specific concepts (algorithm logic, file formats, external systems), anything else in the file that would be unfamiliar.

### Page 3 — Function walkthrough

Cover every function in the file. Group by phase or logical section, one `##` heading per group. For each function, use the pattern:

- **Heading** — function name as inline code, followed inline by a `status` lozenge for its phase.
- **Purpose** — one sentence.
- **Inputs / outputs** — a two-column table (parameter/type/description rows, then a return-value row).
- **Key logic** — prose explaining the main steps, with a `code` macro for any non-obvious snippet.
- **Design decisions** — a `tip` or `note` panel for any choice that deserves explanation (why this approach, not another).

Trivial one/two-line wrapper functions: list in the Page 1 inventory table and describe in one sentence here — no full write-up. Functions containing the core logic deserve the most space. Wrap any long or peripheral group in an `expand` macro so the page stays scannable.

### Page 4 — Line-by-line reference

Dense annotations for the hardest functions — the ones where the code is correct but a reader would still struggle without extra context. Select 3–6 functions maximum across the whole file.

For each selected function:
- A `code` macro (with line numbers on) showing the complete function, or its key section if very long, with numbered inline comment markers (`# ①`, `# ②`, …) matching the source language's comment syntax.
- Immediately below: a table, one row per marker — column 1 the numbered marker, column 2 the explanation.
- Explanations answer "why is this written this way" or "what would go wrong if this line were different" — never just restate the code.
- Note any gotchas, non-obvious side effects, or lines that look wrong but are intentionally written that way.

---

## Guardrails and completion

- **Show key snippets and the hard functions, not the whole file** — representative excerpts, not a full reproduction.
- **Keep Page 4 selective.** 3–6 functions, chosen for difficulty, not coverage.
- **Explain why, not just what.**
- **Use the component library in MACROS.md** for every panel, code block, lozenge, and link — Confluence renders storage format through its own stylesheet, so there's no custom CSS layer to reach for.
- **Validate, get sign-off, then publish — in that order, every time** (mechanics and rationale in PUBLISH.md).

**Done when:** every planned page is drafted, passes the well-formedness check, and is either published with the user's explicit go-ahead or handed to them for review.
