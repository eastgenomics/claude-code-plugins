# claude-code-plugins

Claude Code plugin marketplace for East Genomics.

## Plugins

- **eastgen-plugin** — bundles shared skills, starting with `specification` (generate a full design/build spec for a software project).

## Adding this marketplace

```
/plugin marketplace add eastgenomics/claude-code-plugins
/plugin install eastgen-plugin@claude-code-plugins
```

Or via managed settings:

```json
{
  "extraKnownMarketplaces": {
    "claude-code-plugins": {
      "source": { "source": "github", "repo": "eastgenomics/claude-code-plugins" }
    }
  },
  "enabledPlugins": {
    "eastgen-plugin@claude-code-plugins": true
  }
}
```
