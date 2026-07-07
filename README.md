# claude-code-plugins

Claude Code plugin marketplace for East Genomics.

## Plugins

- **eastgen-plugin** — bundles shared skills, starting with `specification` (generate a full design/build spec for a software project).

## Adding this marketplace

Note: the marketplace is named `eastgenomics`, not `claude-code-plugins` — that name is reserved for official Anthropic marketplaces (`github.com/anthropics/*` only), even though this repo is called `claude-code-plugins`.

```
/plugin marketplace add eastgenomics/claude-code-plugins
/plugin install eastgen-plugin@eastgenomics
```

Or via managed settings:

```json
{
  "extraKnownMarketplaces": {
    "eastgenomics": {
      "source": { "source": "github", "repo": "eastgenomics/claude-code-plugins" }
    }
  },
  "enabledPlugins": {
    "eastgen-plugin@eastgenomics": true
  }
}
```
