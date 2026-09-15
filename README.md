# wrk3-skills

Agent skills for [wrk3](https://github.com/mytmlt/wrk3) — run multiple
branches of the same repo in parallel as git worktrees, each with isolated
ports and a compose project.

This repo was extracted from `wrk3/skills/` so skill updates ship
independently of the CLI. The `wrk3` repo points here instead of vendoring
skills.

Requires `wrk3` CLI v0.1.0+ (see [mytmlt/wrk3](https://github.com/mytmlt/wrk3)).

## Skills

| Skill | Path | Use when |
| ----- | ---- | -------- |
| `wrk3-setup` | `skills/wrk3-setup/SKILL.md` | Check compatibility, propose fixes if incompatible, then author + validate a `wrk3.yaml`. |

## Usage with agents

Copy or symlink the skill dir your agent loads, e.g.:

```bash
# Claude / Codex style (example — adapt to your agent)
cp -r skills/wrk3-setup ~/.agents/skills/
```

Or point your agent at this repo and reference
`skills/wrk3-setup/SKILL.md` directly.

## Versioning

Skills follow the `wrk3` CLI minor they were tested against (see
`skills/<name>/SKILL.md` frontmatter `wrk3-version`). Breaking skill
changes bump the table below.

| skills release | tested wrk3 |
| -------------- | ----------- |
| 0.1.0 | `wrk3 v0.1.0` |

## Contributing

Issues and PRs welcome. Keep skill changes minimal and cite the
`wrk3` CLI behavior they depend on. Update the version table above
when testing against a new CLI release.

## License

MIT — see [LICENSE](LICENSE).
