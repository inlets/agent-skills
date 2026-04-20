# Inlets Agent Skills

[Agent Skills](https://agentskills.io/) for [Inlets](https://inlets.dev) — cloud native tunnels for exposing local and private services to the Internet.

## Skills

| Skill | Description |
|-------|-------------|
| [use-inlets-cloud](skills/use-inlets-cloud/) | Creates and manages inlets cloud tunnels (HTTP and ingress) using the `inlets-pro cloud` CLI. |
| [use-inletsctl](skills/use-inletsctl/) | Creates and manages inlets tunnel exit-servers on cloud VMs using `inletsctl`. |

## Installation

### npx (recommended — works with 40+ agents)

```bash
npx skills add inlets/agent-skills
```

This installs the skill into whichever AI coding agents you have (Claude Code, Amp, Cursor, Codex, Gemini CLI, etc.).

### Manual

Clone and copy the skills into your agent's skills directory:

```bash
git clone https://github.com/inlets/agent-skills.git
cp -r agent-skills/skills/* .claude/skills/   # Claude Code
cp -r agent-skills/skills/* .agents/skills/    # Amp / Codex
cp -r agent-skills/skills/* .cursor/skills/    # Cursor
```

## License

MIT — see [LICENSE](LICENSE).
