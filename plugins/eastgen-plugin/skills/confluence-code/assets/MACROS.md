# Confluence storage-format component library

All page bodies are written in **Confluence storage format** — XHTML with `ac:` (Confluence macro) and `ri:` (resource identifier) namespaced elements. This is what you send as the `body.storage.value` (or `body.representation: storage`) field when publishing — see PUBLISH.md.

Every component below is a drop-in replacement for the equivalent HTML/CSS component in the local `briefing` skill's design system. Use these, not invented markup — Confluence renders storage format through its own stylesheet; you cannot add custom CSS.

---

## Code block

```xml
<ac:structured-macro ac:name="code">
  <ac:parameter ac:name="language">python</ac:parameter>
  <ac:parameter ac:name="title">parse_config()</ac:parameter>
  <ac:parameter ac:name="linenumbers">true</ac:parameter>
  <ac:plain-text-body><![CDATA[def parse_config(path):
    ...
]]></ac:plain-text-body>
</ac:structured-macro>
```

- `language`: use Confluence's supported syntax highlighter names (`python`, `javascript`, `bash`, `sql`, `yaml`, `json`, `xml`, etc.). Omit if the language isn't supported rather than guessing wrong.
- `linenumbers`: `true` for Page 4 (line-by-line reference); optional elsewhere.
- Always wrap the body in `CDATA` — raw `<`/`&` in source code will otherwise break the XML.

---

## Mermaid flowcharts

Confluence has no native Mermaid renderer — a diagram only renders live if the target space has a third-party app for it installed (commonly "Mermaid Charts & Diagrams for Confluence" by weweave, or "Mermaid Macro for Confluence" by SoftwareTao). The macro name differs between apps and even between versions of the same app, so **discover the actual macro name in the target space rather than assuming one.**

### Discover the macro name

Find a page in the target space that already has a rendered diagram, then fetch its storage format with the bundled Confluence MCP tool (see PUBLISH.md) and read off the `ac:name` value used for the diagram macro.

Reuse that exact `ac:name` value and body shape (usually `ac:plain-text-body` with the diagram source in `CDATA`) for every diagram macro on every page you create in this session — the source shape itself is stable, only the macro name and any required parameters vary by app.

### If no example page exists

Use a `code` macro (see Code block above) with `language` set to `mermaid` as a fallback, and tell the user plainly that it will render as plain text, not a live diagram, until a Mermaid app is installed in the space.

### Generic shape once the macro name is known

```xml
<ac:structured-macro ac:name="{discovered-macro-name}">
  <ac:plain-text-body><![CDATA[
flowchart TD
    A[Parse config] --> B{Valid?}
    B -->|yes| C[Run pipeline]
    B -->|no| D[Raise error]
  ]]></ac:plain-text-body>
</ac:structured-macro>
```

Use flowcharts for the same purposes the local `briefing` design system uses SVG diagrams: end-to-end pipelines, phase sequencing, and data-flow shape — most useful on Page 1 (high-level outline) and, for a non-trivial control-flow function, alongside its entry in Page 3.

---

## Panels (callouts)

Four fixed panel types, each with a fixed colour Confluence assigns — pick by meaning, not by desired colour:

| Macro name | Renders as | Use for |
|---|---|---|
| `info` | blue | Neutral context, "in this file" explanations |
| `tip` | green | Positive notes, design decisions that worked out well, "what's done well" |
| `note` | yellow | Caveats, decisions worth flagging, non-obvious behaviour |
| `warning` | red | Gotchas, footguns, "this looks wrong but isn't" |

```xml
<ac:structured-macro ac:name="note">
  <ac:rich-text-body>
    <p><strong>Gotcha:</strong> the retry loop swallows the last exception on purpose — see design-decision panel below.</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

---

## Status lozenge (phase badges)

Replaces the local design's `.fn-phase` pill.

```xml
<ac:structured-macro ac:name="status">
  <ac:parameter ac:name="colour">Purple</ac:parameter>
  <ac:parameter ac:name="title">PHASE 3</ac:parameter>
</ac:structured-macro>
```

Valid `colour` values: `Grey`, `Red`, `Yellow`, `Green`, `Blue`, `Purple`. Pick a consistent colour per phase across all pages in the set (e.g. phase 1 = Blue, phase 2 = Green, phase 3 = Purple, phase 4 = Red, setup/misc = Grey).

---

## Expand (collapsible section)

Use for trivial-function groups or long peripheral content that would otherwise bury the important material.

```xml
<ac:structured-macro ac:name="expand">
  <ac:parameter ac:name="title">Trivial wrapper functions (click to expand)</ac:parameter>
  <ac:rich-text-body>
    <p>...</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

---

## Table of contents

Place once, at the top of the parent/index page only.

```xml
<ac:structured-macro ac:name="toc" />
```

---

## Children display (auto-maintained page list)

Place on the parent/index page. Confluence renders and keeps this current automatically — never hand-write a child-page list.

```xml
<ac:structured-macro ac:name="children" />
```

---

## Page link (for series navigation)

Links resolve by **title**, not ID — you don't need to know a sibling's page ID, only its exact title and space key. This is what makes it possible to draft all pages' navigation panels before any page has been created.

```xml
<ac:link>
  <ri:page ri:content-title="2 · Key concepts" ri:space-key="ENG" />
  <ac:plain-text-link-body><![CDATA[2 · Key concepts]]></ac:plain-text-link-body>
</ac:link>
```

### Series navigation panel (use at the top of every child page)

```xml
<ac:structured-macro ac:name="info">
  <ac:rich-text-body>
    <p><strong>Series:</strong>
      <ac:link><ri:page ri:content-title="{Module} — Code Walkthrough" ri:space-key="ENG"/><ac:plain-text-link-body><![CDATA[Index]]></ac:plain-text-link-body></ac:link> ·
      <ac:link><ri:page ri:content-title="1 · High-level outline" ri:space-key="ENG"/><ac:plain-text-link-body><![CDATA[1 · High-level outline]]></ac:plain-text-link-body></ac:link> ·
      <em>2 · Key concepts</em> ·
      <ac:link><ri:page ri:content-title="3 · Function walkthrough" ri:space-key="ENG"/><ac:plain-text-link-body><![CDATA[3 · Function walkthrough]]></ac:plain-text-link-body></ac:link> ·
      <ac:link><ri:page ri:content-title="4 · Line-by-line reference" ri:space-key="ENG"/><ac:plain-text-link-body><![CDATA[4 · Line-by-line reference]]></ac:plain-text-link-body></ac:link>
    </p>
  </ac:rich-text-body>
</ac:structured-macro>
```

The current page renders as plain `<em>` text (no link to itself); every sibling and the parent are links. Update this panel on every page — copy-paste and swap the `<em>` position.

---

## Tables

Plain storage-format tables — no macro needed, used for function inventories, I/O grids, dependency lists, and annotation lists (marker number | explanation).

```xml
<table>
  <tbody>
    <tr><th>Parameter</th><th>Type</th><th>Description</th></tr>
    <tr><td>path</td><td>str</td><td>Absolute path to the config file</td></tr>
  </tbody>
</table>
```

---

## Inline code

Standard `<code>` inline, e.g. `<code>parse_config()</code>` in a heading or paragraph — no macro needed.

---

## Staying within the macro set

Confluence renders storage format through its own stylesheet — reach for the nearest native macro rather than replicating the local HTML design system:

| Local design system | Confluence equivalent |
|---|---|
| Sidebar TOC | `toc` macro |
| Hero header | Page title + opening paragraph |
| Stat strip / card grid | A table |
| Hand-maintained child-page list | `children` macro |

Custom `<div class="...">` markup renders as plain, unstyled text — Confluence strips classes it doesn't recognise.
