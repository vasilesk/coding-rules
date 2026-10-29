# CLAUDE.md

This repository is the `coding-rules` Claude Code plugin with Agent Skills in the [Agent Skills format](https://agentskills.io/specification). It is listed in the `vasilesk` marketplace, which lives in a separate repository: [vasilesk/agent-skills](https://github.com/vasilesk/agent-skills).

## Layout

- `skills/<name>/SKILL.md` — one skill per directory; `name` in the frontmatter matches the directory name.
- `skills/<name>/references/` — optional detail loaded on demand; link it from `SKILL.md` and say when to read it.
- `.claude-plugin/plugin.json` — plugin manifest; the single source of the version. The marketplace entry does not set a version.

## Writing skills

- `description` is the only text read before a skill is loaded: state what the skill covers, then when to use it, with concrete terms. Quote it — a `: ` inside breaks YAML silently.
- Do not describe steps of a workflow in `description`, or the agent acts on it and skips the body.
- Each rule lives in one skill; others reference it by name (`go-coding-conventions` refers to `coding-conventions` instead of repeating it).
- Give the reason with the rule, so the agent can apply it to cases the rule does not name.
- Keep `SKILL.md` under ~250 lines; move long examples to `references/`.

## Releasing

- Bump `version` in `.claude-plugin/plugin.json` with every change. Claude Code caches installed copies per version and compares the version as a plain string, not by semver: any change of the string delivers an update, and without a bump users keep the old copy.
- Pick the part to bump by the size of the change, as a signal to readers: patch — wording, or refining, narrowing or relaxing existing rules (new bullets included); minor — a new skill or a new area of rules; major — a renamed or removed skill.
- Run `claude plugin validate .` before pushing.
- Users get the release from this repository's default branch: the marketplace points here without pinning a ref, so nothing changes in `vasilesk/agent-skills`.
