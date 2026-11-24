# Skills

My personal agent skills, packaged for the [`skills`](https://skills.sh) CLI (`npx skills`), so they can be installed on any machine.

## Skills

| Skill | What it does |
| --- | --- |
| `commit` | Groups the current changes into logical commits with Conventional Commits messages. |
| `eod-slack` | Builds an end-of-day summary from Clockify entries and ClickUp tasks, for posting to Slack. |
| `new-task` | Turns a task reference into a ClickUp brief, an investigation, and an implementation plan in ISO/IEC/IEEE 29148 form (Laravel projects). |
| `update-dependencies` | Updates Composer and Node.js dependencies, bumps manifests to the installed versions, reports changes, and plans upgrades to available majors. |

## Install

All skills, globally:

```bash
npx skills add wilsenhc/skills -g --all
```

One skill:

```bash
npx skills add wilsenhc/skills --skill commit -g
```

From a local checkout:

```bash
npx skills add . -g --all
```

## Requirements

- `eod-slack` expects the Clockify and ClickUp MCP tools to be connected. It reads the Clockify key from `~/.config/opencode/opencode.jsonc` or `$CLOCKIFY_API_KEY`, and the ClickUp space for the pending-work check from `$CLICKUP_EOD_SPACE_ID`.
- `new-task` expects ClickUp MCP (optional, falls back to user input), a Laravel project, and browser tooling for reproduction steps.
- `commit` writes commits only when explicitly invoked, never pushes, and uses Conventional Commits.
- `update-dependencies` expects Composer and/or a Node.js project using npm, bun, or pnpm.

## Security

Skills run with the permissions of the agent that loads them. `commit` writes
commits only on an explicit request and never pushes. `eod-slack` and `new-task`
treat ClickUp and Clockify content as untrusted data, never as instructions.
`new-task` uses additive migrations only and asks before any database cleanup.
`update-dependencies` treats changelogs and registry output as untrusted data,
runs only project-local updates, and never installs majors without approval.

## Layout

```
skills/
├── commit/
│   ├── SKILL.md
│   └── agents/
│       └── openai.yaml
├── eod-slack/
│   ├── SKILL.md
│   └── agents/
│       └── openai.yaml
├── new-task/
│   ├── SKILL.md
│   └── agents/
│       └── openai.yaml
└── update-dependencies/
    ├── SKILL.md
    └── agents/
        └── openai.yaml
```

Each skill is a directory with a `SKILL.md` and YAML frontmatter. The directory name matches the frontmatter `name`. Codex reads the optional `agents/openai.yaml` for its UI name and invocation policy; Claude Code and OpenCode read `disable-model-invocation` from the frontmatter.

`new-task` is explicit-only on every harness: `disable-model-invocation: true` and `metadata.opencode/autoinvoke: false` in its frontmatter, plus `policy.allow_implicit_invocation: false` in its Codex metadata. `commit`, `eod-slack` and `update-dependencies` use the default auto-invocation everywhere.

## License

MIT — see [LICENSE](LICENSE).
