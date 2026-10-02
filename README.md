# Inlets Agent Skills

[Agent Skills](https://agentskills.io/) for [Inlets](https://inlets.dev) — cloud native tunnels for exposing local and private services to the Internet.

## Skills

| Skill | Description |
|-------|-------------|
| [setup-uplink](skills/setup-uplink/) | Installs Inlets Uplink on Kubernetes with Traefik, other ingress controllers, or Istio, supporting private tunnels and optional public HTTPS exposure. |
| [use-inlets-cloud](skills/use-inlets-cloud/) | Creates and secures hosted HTTP and ingress tunnels with generated or custom domains using the `inlets-pro cloud` CLI. |
| [use-inlets-operator](skills/use-inlets-operator/) | Installs and operates inlets-operator lifecycle management for Kubernetes LoadBalancer Services and cloud tunnel infrastructure. |
| [use-inlets-pro](skills/use-inlets-pro/) | Configures and secures standalone Inlets Pro TCP and automated HTTPS tunnels, including authentication and DNS setup. |
| [use-inlets-uplink](skills/use-inlets-uplink/) | Creates and manages Uplink tunnels through Kubernetes CRDs, the CLI, or REST APIs, including self-signed TLS trust and end-to-end verification. |
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

## All-in-one Uplink installation

Use `setup-uplink` to install the control plane, configure TLS, and verify the management API on an existing cluster. Example prompt:

```text
Set up Inlets Uplink on the Kubernetes cluster <cluster-name>.
Use the setup-uplink skill and the kubeconfig for that cluster. Create
values.yaml in the current working directory. Use self-signed certificates
and keep tunneled applications private. The Uplink license is available
at ~/.inlets/LICENSE_UPLINK.
```

Results of private Uplink setup with self-signed TLS using **OpenCode**:

| Model | Average setup time |
|---|---:|
| GPT-5.6 Luna (low reasoning) | 4m 14s |
| Qwen 3.8 27B | 3m 36s |

## License

MIT — see [LICENSE](LICENSE).
