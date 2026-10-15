# instabox-skill

An [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills) that teaches Claude Code (and other skill-aware agents) to work with [instabox](https://instabox.sh) cloud Linux machines:

- the `instabox` CLI: create, SSH, exec, pause/resume, ports, snapshots, Vault, API keys
- the `@insforge/instabox-sdk` TypeScript SDK
- the HTTP API

Unofficial, community-maintained. Based on https://instabox.sh/docs.

## Install

**Claude Code plugin**

```
/plugin marketplace add igzard/instabox-skill
/plugin install instabox@instabox-skill
```

**skills CLI** (Claude Code, Codex, Cursor, ...)

```sh
npx skills add igzard/instabox-skill
```

**Manual**

```sh
git clone https://github.com/igzard/instabox-skill
cp -r instabox-skill/skills/instabox ~/.claude/skills/
```

> `npx instabox@latest` installs its own official `instabox` skill into `~/.claude/skills/instabox`. The manual copy above overwrites it; use the plugin install to keep both.

## Requirements

The `instabox` CLI (`curl -fsSL https://instabox.sh/install.sh | sh`) and a sign-in (`instabox login`) or an API key (`INSTABOX_API_KEY=ibk_...`).

## Layout

```
skills/instabox/
  SKILL.md                  CLI, lifecycle, ports, SSH, snapshots, Vault, keys, limits
  references/sdk.md         TypeScript SDK
  references/http-api.md    HTTP API
```

## Try it

Ask your agent: *"Start an instabox machine, clone my repo into it and run the tests."*

## Contributing

instabox moves fast. If the skill disagrees with `instabox --help` or the docs, open a PR.

## License

MIT
