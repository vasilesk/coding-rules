# coding-rules

Coding rules as Agent Skills: general programming conventions, Go coding conventions, and a hexagonal project layout sliced by bounded contexts. Installable as a Claude Code plugin from the [`vasilesk` marketplace](https://github.com/vasilesk/agent-skills); works with any tool that supports the [Agent Skills](https://agentskills.io) format.

## Skills

| Skill | What it covers |
| --- | --- |
| [`coding-conventions`](skills/coding-conventions/SKILL.md) | Language-agnostic rules: immutable objects and validating constructors, typed identifiers, comment content, structured logging, cursor pagination, database and configuration conventions |
| [`go-coding-conventions`](skills/go-coding-conventions/SKILL.md) | Go-specific rules on top of `coding-conventions`: receivers, `mo.Option`, `lo`, injected clock, error wrapping and sentinels, iterators for batch processing, testify and gomock, file layout |
| [`go-hexagonal-architecture`](skills/go-hexagonal-architecture/SKILL.md) | Hexagonal architecture (ports and adapters) sliced by bounded context: directory layout, dependency rules, `kernel`/`platform`/`pkg`, ports, data formats, wiring, enforcement with go-arch-lint and depguard |

`go-coding-conventions` builds on `coding-conventions` and refers to `go-hexagonal-architecture` for architecture rules, so install them together.

## Installation

### Claude Code

```
/plugin marketplace add vasilesk/agent-skills
/plugin install coding-rules@vasilesk
```

Or from the terminal:

```bash
claude plugin marketplace add vasilesk/agent-skills
claude plugin install coding-rules@vasilesk
```

Restart the session afterwards. Skills appear as `coding-rules:coding-conventions` and so on.

**Updating:** run `/plugin marketplace update vasilesk`, then update the plugin from the `/plugin` menu (or `claude plugin update coding-rules@vasilesk`). To get updates automatically, enable auto-update for the `vasilesk` marketplace in the `/plugin` menu.

**Migrating from `agent-skills@vasilesk`:** the plugin was renamed. Run `claude plugin uninstall agent-skills@vasilesk`, update the marketplace, and install `coding-rules@vasilesk`.

### Other tools (Codex, Cursor, Gemini CLI, Copilot, ...)

With the [skills](https://skills.sh) CLI:

```bash
npx skills add https://github.com/vasilesk/coding-rules --all
```

Or copy (or symlink) the directories from `skills/` into your tool's skills directory, e.g. `~/.claude/skills/` for Claude Code without the plugin system.

## Development

Load the plugin straight from the working copy, bypassing the plugin cache:

```bash
claude --plugin-dir .
```

Bump `version` in `.claude-plugin/plugin.json` with every change you want installed copies to pick up: installed plugins are cached per version.

## License

[MIT](LICENSE)
