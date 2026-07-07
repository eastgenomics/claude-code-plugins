# Publishing to Confluence

## Use the bundled MCP server

This plugin bundles the official Atlassian remote MCP server (Confluence + Jira Cloud), authenticated per user via OAuth — no shared credentials or tokens live in this plugin. Discover its tools:

```
ToolSearch: query "confluence"
```

Use these tools directly to create and update pages — they handle auth, page lookups, and versioning for you. If a call reports the server needs authentication, that user hasn't signed in yet: direct them to run `/mcp` (interactive) or `claude mcp login atlassian` to complete their own one-time OAuth flow.

## Drafting workflow (do this before calling any tool)

1. Write each page's storage-format body to a local scratch file first, e.g. `./confluence-drafts/{page-slug}.xml`, so it stays readable and reviewable.
2. **Validate** every draft is well-formed before publishing:

```bash
python3 -c "
import xml.etree.ElementTree as ET
NS = '''<root xmlns:ac=\"http://a\" xmlns:ri=\"http://b\">{}</root>'''
body = open('confluence-drafts/1-high-level-outline.xml').read()
try:
    ET.fromstring(NS.format(body))
    print('OK: well-formed')
except Exception as e:
    print(f'INVALID: {e}')
"
```

   This catches unescaped `&`/`<` in code snippets (the single most common failure — always run source snippets through XML escaping or wrap them in `CDATA` via the `code` macro's `plain-text-body`, never inline).

3. **Show the user the page plan** (titles, hierarchy, one-paragraph summary of each page) and get explicit go-ahead before creating anything. If they ask to see full drafts first, show them — these become visible to the whole space once published, so there's no "publish first, fix later" step to lean on.

## Confirmation gate

Before calling any page-creation or page-update tool:

1. The user has confirmed the space (and parent page, if applicable).
2. The user has seen the page plan (titles + hierarchy + one-line summary each) and said to proceed.
3. Every draft has passed the well-formedness check above.

Creating a page is a visible, shared-state action — treat it with the same care as pushing a commit or opening a PR, not as a reversible local edit.
