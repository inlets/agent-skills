---
name: use-inlets-uplink
description: Creates, connects, inspects, updates, and removes Inlets Uplink tunnels through the Kubernetes Tunnel CRD, inlets-pro tunnel CLI plugin, or REST API. Use for TCP and HTTP upstreams, tenant namespaces, client deployment, optional self-signed TLS trust, tunnel verification, Uplink API authentication, entitlement, or billing-capacity questions.
---

# Use Inlets Uplink

Manage private customer endpoints from an Inlets Uplink control plane. Prefer the interface the user already operates: CRDs for GitOps, the CLI for interactive work, and REST for application integration.

For control-plane installation, ingress, and certificate issuance, use [setup-uplink](../setup-uplink/SKILL.md), or its [upstream copy](https://github.com/inlets/agent-skills/blob/master/skills/setup-uplink/SKILL.md) if not installed. When continuing from that skill, reuse the supplied hostname/port, tenant scope, mesh injection policy, visibility, and any public CA/certificate file.

Docs: https://docs.inlets.dev/uplink/

## Prepare a tenant namespace

Do not create tunnels in the `inlets` control-plane namespace. Create or select a labeled tenant namespace and ensure its `inlets-uplink-license` secret exists.

```bash
export TUNNEL_NS="tenant-name"
kubectl get namespace "$TUNNEL_NS" >/dev/null 2>&1 ||
  kubectl create namespace "$TUNNEL_NS"
kubectl label namespace "$TUNNEL_NS" inlets.dev/uplink=1 --overwrite
```

During discovery, inspect only Secret names and metadata. Never decode, print, preview, or copy a license, API key, or tunnel token, even partially. Prefer secret references, restrictive token files, or environment-variable flags when a credential is actually required. For repeatable GitOps, pre-create a strong token Secret and set `spec.tokenRef`; an operator-generated token changes if the Tunnel is deleted and recreated.

For the documented Istio sidecar setup, enable the selected mesh injection policy on the tenant namespace before creating workloads. Respect any `tunnelsNamespace` restriction from the Uplink installation. Provision a missing tenant license Secret from the administrator's protected license file or existing secret-management workflow; do not ask for its value in chat.

## Choose a management interface

- Use a `Tunnel` CRD (`uplink.inlets.dev/v1alpha1`) for declarative Kubernetes and GitOps.
- Use `inlets-pro plugin get tunnel` once to install/update the tunnel plugin, then `inlets-pro tunnel ...` for interactive management.
- Use the REST API for SaaS portals and automation. Read [references/api-and-billing.md](references/api-and-billing.md) before implementing any API or billing request. Never infer an endpoint or authentication method.

Discover current schemas and flags instead of assuming a pinned version:

```bash
kubectl explain tunnels.uplink.inlets.dev.spec
inlets-pro tunnel create --help
inlets-pro uplink client --help
```

Always qualify the Uplink resource as `tunnels.uplink.inlets.dev`. Some clusters also install the inlets-operator `tunnels.operator.inlets.dev` CRD, so unqualified `kubectl get tunnel` and similar commands may inspect or mutate the wrong resource.

## Build tunnels incrementally

Read [references/tunnel-patterns.md](references/tunnel-patterns.md) for complete CRD, CLI, and client examples. Start with the smallest mapping, verify it, and only then add upstreams.

1. For one TCP service, declare one `tcpPorts` entry and connect it with `--upstream PUBLIC_PORT=PRIVATE_HOST:PRIVATE_PORT`.
2. For multiple TCP services, add every public port to `tcpPorts` and repeat `--upstream` on the client.
3. For one HTTP service, use the tunnel's default HTTP service and a single URL upstream.
4. For multiple HTTP services, add stable `upstreamDomains` and map each generated hostname to one URL upstream.
5. For mixed TCP and HTTP, combine `tcpPorts` and `upstreamDomains` in one Tunnel and pass both mapping forms to one client.

The declared public TCP port must match the left side of its client mapping. For multiple HTTP upstreams, each `upstreamDomains` entry is a short name such as `gateway`. Uplink creates a Kubernetes Service for each entry, addressed as `<name>.<tunnel-namespace>` such as `gateway.tunnels`. Use that address as the client mapping key and as the HTTP `Host` when selecting the upstream. These are internal Kubernetes service addresses, not public Internet DNS.

## Connect and verify

Choose a client host that can reach both the ingress and upstream, reusing a compatible installed runtime when possible. Run management commands in an environment with appropriate cluster access; the runtime client itself needs neither a kubeconfig nor the management plugin. Read [references/client-verification.md](references/client-verification.md) when deploying a test client or running an end-to-end check.

Let the plugin derive the deployed router URL, but never print its raw output: `tunnel connect` embeds the tunnel token in CLI, systemd, and Kubernetes formats. Capture it in a restrictive temporary file, extract only the URL, and retrieve the token separately into another restrictive file:

```bash
export TUNNEL_NAME="customer-edge"
export UPLINK_DOMAIN="uplink.example.com"
umask 077
CONNECT_FILE="$(mktemp)"
TOKEN_FILE="$(mktemp)"
trap 'rm -f "$CONNECT_FILE" "$TOKEN_FILE"' EXIT

inlets-pro tunnel connect "$TUNNEL_NAME" \
  --namespace "$TUNNEL_NS" --domain "$UPLINK_DOMAIN" \
  --upstream http://127.0.0.1:8080 --quiet > "$CONNECT_FILE"
inlets-pro tunnel token "$TUNNEL_NAME" \
  --namespace "$TUNNEL_NS" > "$TOKEN_FILE"

UPLINK_URL="$(
  sed -nE 's/.*--url=?([^[:space:]\\]+).*/\1/p' "$CONNECT_FILE" \
  | head -n1
)"
test -n "$UPLINK_URL"
```

Keep the derived URL and protected token file available to the client for as long as needed. The example's variables are shell-local and its EXIT trap deletes the files when that shell ends: either launch the temporary client in that shell or securely persist the required values/files before it exits. For a remote or persistent client, provision a protected token file on the client host and pass its path there. If `UPLINK_DOMAIN` is unknown, ask the Uplink administrator or inspect only Ingress hostnames with `kubectl get ingress -n inlets`; do not search Secrets or container environment values for it. Keep stdout from `tunnel connect` separate from stderr and never display the captured stdout, including during error handling.

Do not hard-code a router path such as `/tunnels/<name>` or `/<namespace>/<name>`; the deployed router and plugin determine it. Never display `$CONNECT_FILE`, `$TOKEN_FILE`, or even a credential prefix. Run the client with `--url "$UPLINK_URL"` and `--token-file "$TOKEN_FILE"`; never put a token in command arguments. Treat generated systemd or Kubernetes output as secret material because it may embed `--token`; redirect it to a restrictive file and replace the embedded token with an environment, credential, or Secret reference before displaying, applying, or installing it.

### Optional trust for self-signed or private certificates

For a self-signed evaluation or private CA, inspect `inlets-pro uplink client --help` for `--tls-ca`. Obtain the public PEM trust bundle from the Uplink administrator or the `setup-uplink` handoff and make it readable on the client host. Set `UPLINK_CA_FILE` to its actual path and add the flag before the first connection:

```bash
inlets-pro uplink client \
  --url "$UPLINK_URL" \
  --token-file "$TOKEN_FILE" \
  --tls-ca "$UPLINK_CA_FILE" \
  --upstream http://127.0.0.1:8080
```

Preserve the securely derived URL and chosen upstream mappings; `--tls-ca` is optional for certificates already trusted by the client's normal trust store. It verifies the Uplink server connection, not TLS at an HTTPS upstream, and it does not replace the tunnel token. The URL hostname must resolve from the client and match the certificate's DNS names. Obtain the certificate over a trusted administrative channel; never transfer its private key or fetch trust blindly from an unauthenticated endpoint.

For a directly self-signed certificate, trust the administrator-provided server certificate; for private PKI, use its CA bundle. Do not disable verification to bypass an unknown-authority error. If the runtime lacks `--tls-ca`, use a compatible version that supports it. For systemd or Kubernetes clients, retain the flag and provision the PEM at a durable, readable path (a read-only mount for a Pod). A reissued self-signed certificate may require updating that file. Keep local self-signed evaluations distinct from production PKI.

### Check the tunnel and data path

Verify both control-plane state and each data path:

```bash
kubectl get tunnels.uplink.inlets.dev -n "$TUNNEL_NS"
kubectl describe tunnels.uplink.inlets.dev "$TUNNEL_NAME" -n "$TUNNEL_NS"
kubectl get pods,svc -n "$TUNNEL_NS"
```

Do not assume the Tunnel CR exposes a `Ready` condition. Verify that its status contains the expected Deployment, Service, and token references, inspect sync events, then use `kubectl rollout status deployment/"$TUNNEL_NAME" -n "$TUNNEL_NS" --timeout=60s`.

Never guess a generated Service name, DNS name, port, or Host value. Inspect the generated Services first with `kubectl get svc -n "$TUNNEL_NS"` and use the observed values. Test TCP with the protocol's native client or `nc`; test HTTP with `curl --fail --max-time 10`, setting the observed Host value when routing multiple HTTP upstreams.

For temporary validation, keep credentials out of command arguments, use time-bounded checks, and ensure the client and test resources are stopped or removed afterward. Use the observed Tunnel Service control endpoint for an in-cluster test, or the securely derived external `$UPLINK_URL` for a client outside the Uplink cluster.

## Operate safely

Use `inlets-pro tunnel list`, `token`, `rotate`, and `remove`, or the equivalent fully qualified `kubectl` operations. Inspect events, server/client logs, hostname mappings, and whether the client URL uses `ws` versus `wss` when troubleshooting. Never run broad process commands such as `ps -ef` or `pgrep -af` when clients may have been started with legacy `--token` arguments; inspect a known PID or service unit without printing its arguments. Use `--demux` only when needed for problematic or high-throughput TCP workloads and validate its trade-offs against the deployed version's documentation.

When the question concerns license quantity, metering, entitlement, or changing billable tunnel capacity, read the billing section in [references/api-and-billing.md](references/api-and-billing.md) before taking any action. The billing quantity endpoint is exactly `GET /v1/billing/quantity`; it requires the separately issued billing API key and an administrator-provided billing API base URL. Never substitute the product license, client API token, or tunnel token, and never invent a likely billing route. Require explicit authorization before `PUT /v1/billing/quantity`.
