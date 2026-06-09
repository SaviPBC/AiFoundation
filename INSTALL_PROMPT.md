# Install Prompt

Users paste the text below into Claude Code to install the plugin. No other steps required.

---

Please set up the Savi AI Foundation plugin on my Claude Code. Do the following:

1. Read `~/.claude/settings.json` (on Windows: `C:\Users\<your-username>\.claude\settings.json`). If it doesn't exist, start with an empty JSON object `{}`.

2. Merge in this configuration — preserve any existing keys, only add what's missing:

```json
{
  "extraKnownMarketplaces": {
    "savi-ai-foundation": {
      "source": {
        "source": "github",
        "repo": "SaviPBC/AiFoundation"
      },
      "autoUpdate": true
    }
  },
  "enabledPlugins": {
    "savi-ai-foundation@savi-ai-foundation": true
  }
}
```

3. If `~/.claude/CLAUDE.md` exists and contains Savi development workflow instructions (look for "Savi Development Workflow" near the top), remove that content. If it was the only content in the file, delete the file entirely.

4. Save the updated `settings.json`.

5. Confirm when done, and tell me to restart Claude Code to activate the plugin.