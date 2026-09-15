# Claude Marketplace

A [Claude Code plugin marketplace](https://docs.claude.com/en/docs/claude-code/plugin-marketplaces) maintained by Cealtea.

## Install the marketplace

In Claude Code:

```text
/plugin marketplace add Cealtea/claude-marketplace
```

Then browse and install plugins:

```text
/plugin install work-recap@cealtea
```

Or from the shell:

```sh
claude plugin marketplace add Cealtea/claude-marketplace
claude plugin install work-recap@cealtea
```

## Plugins

| Plugin | Description |
| --- | --- |
| `work-recap` | Compiles completed Linear tickets and closed Obsidian daily-note todos into a recap and DMs it to you on Slack. Run `/work-recap:work-recap`. Requires the Linear and Slack MCP connectors. On first run it asks for your Slack and Linear user IDs, Obsidian folder, and time zone, and saves them to `~/.config/work-recap/config.json`. Edit or delete that file to reconfigure. |

## Repo layout

```text
.claude-plugin/marketplace.json   # marketplace manifest, lists every plugin
plugins/<plugin-name>/            # one directory per plugin
  .claude-plugin/plugin.json      # plugin manifest
  skills/<skill>/SKILL.md         # skills (also: commands/, agents/, hooks/)
```

## Adding a plugin

1. Create `plugins/<name>/.claude-plugin/plugin.json` with at least `name`, `description`, and `version`.
2. Add skills, commands, agents, or hooks under that plugin directory.
3. Add an entry to the `plugins` array in `.claude-plugin/marketplace.json` with `"source": "./plugins/<name>"`.
4. Validate before committing:

```sh
claude plugin validate .
```

## Local development

Test a plugin without publishing:

```sh
claude --plugin-dir ./plugins/work-recap
```

Or add this checkout as a marketplace directly:

```text
/plugin marketplace add /path/to/claude-marketplace
```
