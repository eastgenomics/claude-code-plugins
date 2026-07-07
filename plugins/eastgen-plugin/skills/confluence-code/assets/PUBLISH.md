# Publishing to Confluence

## Prefer the bundled MCP server

This plugin bundles the official Atlassian remote MCP server (Confluence + Jira Cloud, OAuth-authenticated per user — no shared credentials). Check for it first:

```
ToolSearch: query "confluence"
```

If Atlassian tools are available, use them directly for creating and updating pages instead of the curl workflow below — they handle auth, storage-format encoding, and page-ID/version lookups for you. On first use for a given user, a tool call may report the server needs authentication; direct them to run `/mcp` (interactive) or `claude mcp login atlassian` to complete their own one-time OAuth sign-in.

The REST-via-curl workflow below is the fallback for environments without the MCP server connected (e.g. Server/Data Center instances the official server doesn't cover, or a session where the user prefers a static API token).

## Configuration (curl fallback only)

Source these from the environment — keep tokens out of chat entirely.

| Variable | Purpose |
|---|---|
| `CONFLUENCE_BASE_URL` | e.g. `https://yourteam.atlassian.net` (Cloud) or `https://confluence.yourcompany.com` (Server/Data Center) |
| `CONFLUENCE_EMAIL` | Cloud only — the account email for basic auth |
| `CONFLUENCE_API_TOKEN` | Cloud: API token used as the basic-auth password. Server/DC: a Personal Access Token, used as a Bearer token instead |
| `CONFLUENCE_SPACE_KEY` | Target space, e.g. `ENG` — confirm with the user if not already established |

If a required variable is missing, stop and ask the user to set it.

**Auth header:**
- Cloud: `-u "$CONFLUENCE_EMAIL:$CONFLUENCE_API_TOKEN"` (curl basic auth)
- Server/DC: `-H "Authorization: Bearer $CONFLUENCE_API_TOKEN"`

## Drafting workflow (do this before touching the API)

1. Write each page's storage-format body to a local scratch file first, e.g. `./confluence-drafts/{page-slug}.xml`, so it stays readable and reviewable — then reference that file from curl rather than inlining the body.
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

## Creating pages (REST API v1 — `/wiki/rest/api/content`)

Works the same shape on Cloud (prefix `/wiki`) and Server/DC (no `/wiki` prefix — just `/rest/api/content`); adjust `CONFLUENCE_BASE_URL` accordingly for the deployment type.

**1. Create the parent/index page:**

```bash
jq -n --arg title "{Module} — Code Walkthrough" \
      --arg space "$CONFLUENCE_SPACE_KEY" \
      --rawfile body confluence-drafts/0-index.xml \
      '{type:"page", title:$title, space:{key:$space}, body:{storage:{value:$body, representation:"storage"}}}' \
  > /tmp/create-parent.json

curl -sf -u "$CONFLUENCE_EMAIL:$CONFLUENCE_API_TOKEN" \
  -X POST "$CONFLUENCE_BASE_URL/wiki/rest/api/content" \
  -H "Content-Type: application/json" \
  --data @/tmp/create-parent.json | tee /tmp/parent-response.json | jq -r '.id, ._links.webui'
```

Capture the returned `id` — you need it as the `ancestors` parent ID for every child (page *links* resolve by title, but page *creation* still needs the parent's numeric ID).

**2. Create each child page**, same shape, adding `ancestors`:

```bash
PARENT_ID=$(jq -r '.id' /tmp/parent-response.json)

jq -n --arg title "1 · High-level outline" \
      --arg space "$CONFLUENCE_SPACE_KEY" \
      --arg parent "$PARENT_ID" \
      --rawfile body confluence-drafts/1-high-level-outline.xml \
      '{type:"page", title:$title, space:{key:$space}, ancestors:[{id:$parent}], body:{storage:{value:$body, representation:"storage"}}}' \
  > /tmp/create-child-1.json

curl -sf -u "$CONFLUENCE_EMAIL:$CONFLUENCE_API_TOKEN" \
  -X POST "$CONFLUENCE_BASE_URL/wiki/rest/api/content" \
  -H "Content-Type: application/json" \
  --data @/tmp/create-child-1.json | jq -r '.id, ._links.webui'
```

Repeat for pages 2, 3 (and 3b/3c/… if split), 4. Since navigation panels link by title+space (not ID), all pages' bodies can be fully drafted up front — no need to sequence drafting around ID availability, only creation needs the parent ID.

**3. Print every page's web URL** (`_links.webui`, relative — prefix with `CONFLUENCE_BASE_URL`) at the end so the user has direct links to review.

## Updating an existing page

Requires the current `version.number` — fetch it first, then increment:

```bash
CURRENT_VERSION=$(curl -sf -u "$CONFLUENCE_EMAIL:$CONFLUENCE_API_TOKEN" \
  "$CONFLUENCE_BASE_URL/wiki/rest/api/content/$PAGE_ID?expand=version" | jq -r '.version.number')

jq -n --arg title "1 · High-level outline" \
      --argjson version "$((CURRENT_VERSION + 1))" \
      --rawfile body confluence-drafts/1-high-level-outline.xml \
      '{type:"page", title:$title, version:{number:$version}, body:{storage:{value:$body, representation:"storage"}}}' \
  > /tmp/update-child-1.json

curl -sf -u "$CONFLUENCE_EMAIL:$CONFLUENCE_API_TOKEN" \
  -X PUT "$CONFLUENCE_BASE_URL/wiki/rest/api/content/$PAGE_ID" \
  -H "Content-Type: application/json" \
  --data @/tmp/update-child-1.json | jq -r '.id, ._links.webui'
```

Confluence rejects an update whose `version.number` doesn't match current+1 — always re-fetch immediately before updating rather than reusing a version number from earlier in the session.

## Confirmation gate

Before any `curl -X POST` or `-X PUT` against the Confluence API:

1. The user has confirmed the space (and parent page, if applicable).
2. The user has seen the page plan (titles + hierarchy + one-line summary each) and said to proceed.
3. Every draft has passed the well-formedness check above.

Creating a page is a visible, shared-state action — treat it with the same care as pushing a commit or opening a PR, not as a reversible local edit.
