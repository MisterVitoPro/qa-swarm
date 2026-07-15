# Repository guidance

- Maintain QA Swarm as a dual-client plugin for Claude Code and Codex.
- Keep `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json` on the same semantic version.
- Keep skill frontmatter compatible with Codex: only supported fields, and each skill name must match its folder name.
- Keep orchestration host-neutral. Document Claude slash commands and Codex `$` skill mentions at user-facing invocation points.
- Resolve bundled agent definitions relative to the active `SKILL.md`; Codex does not register files under `agents/` as named agents automatically.
- Run both plugin validators and the Codex skill-frontmatter validation before releasing.
- Release with an immutable plain `v<version>` tag, then update both catalogs in `qa-claude-market` to that tag and commit SHA.
