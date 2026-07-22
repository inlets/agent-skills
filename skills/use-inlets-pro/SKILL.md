---
name: use-inlets-pro
description: Configures, runs, secures, and troubleshoots standalone Inlets Pro HTTP and TCP tunnel servers and clients. Use for raw TCP forwarding, single or multiple HTTP upstreams, automated TLS/HTTPS with DNS, shared-token control-plane authentication, HTTP Basic/Bearer/OAuth protection, IP filtering, systemd, or Kubernetes manifests.
---

# Use Inlets Pro

Build a tunnel as a server/client pair: the public server exposes the data plane and accepts an outbound WebSocket connection from the private client, which forwards traffic to local upstreams.

Docs: https://docs.inlets.dev/

## Inspect the installed version

Run `inlets-pro version` and the relevant `--help` commands before composing production commands. Flags and supported OAuth providers differ between releases.

Generate a strong shared tunnel token and keep it out of process listings where practical:

```bash
openssl rand -base64 32 > inlets-token.txt
```

Use the same token on both ends through `--token-file`, `--token-env`, or `--token`. Prefer file or environment input and restrictive file permissions. The shared token authenticates the control-plane client; it does not protect visitors to an exposed HTTP service.

## TCP tunnel

On the public server, include the public IP or DNS name in the automatically generated control-plane certificate:

```bash
inlets-pro tcp server \
  --auto-tls \
  --auto-tls-san PUBLIC_IP_OR_NAME \
  --token-file ./inlets-token.txt
```

On the private network, publish one or more TCP ports and select their upstream host:

```bash
inlets-pro tcp client \
  --url wss://PUBLIC_IP_OR_NAME:8123 \
  --token-file ./inlets-token.txt \
  --upstream 127.0.0.1 \
  --ports 22,5432
```

`--upstream` applies to the published ports. If public and private ports differ, or targets differ, inspect client-forwarding/local mapping support with `inlets-pro tcp client --help`; do not invent a mapping syntax. Restrict public source addresses with repeated server `--allow-ips` values when appropriate. Enable Proxy Protocol only when the upstream explicitly supports the selected version.

## Automated HTTPS tunnel

Use HTTP mode with `--letsencrypt-domain` to terminate public HTTPS on ports 80 and 443 through Let's Encrypt HTTP-01. Read [references/https-and-dns.md](references/https-and-dns.md) before provisioning or changing DNS.

```bash
inlets-pro http server \
  --auto-tls \
  --auto-tls-san PUBLIC_IP_OR_NAME \
  --letsencrypt-domain app.example.com \
  --token-file ./inlets-token.txt
```

Start with one upstream:

```bash
inlets-pro http client \
  --url wss://PUBLIC_IP_OR_NAME:8123 \
  --token-file ./inlets-token.txt \
  --upstream app.example.com=http://127.0.0.1:3000
```

Then add repeatable domain/upstream mappings only after the first works:

```bash
inlets-pro http client \
  --url wss://PUBLIC_IP_OR_NAME:8123 \
  --token-file ./inlets-token.txt \
  --upstream app.example.com=http://127.0.0.1:3000 \
  --upstream metrics.example.com=http://192.168.1.20:9090
```

Pass every public hostname to the server using repeated `--letsencrypt-domain` flags, unless the installed help explicitly documents a different format. Use the staging issuer while iterating to avoid Let's Encrypt production rate limits.

## Authentication options

- Control plane: use the same shared `--token-file`, `--token-env`, or `--token` on server and client.
- HTTP data plane: configure Basic auth with `--basic-auth-username` and `--basic-auth-password` on the HTTP client.
- HTTP data plane: configure an API-friendly Bearer token with `--bearer-token`; combine it with Basic or OAuth when needed.
- HTTP data plane: configure OAuth with the provider, client ID/secret, callback `https://DOMAIN/_/oauth/callback`, and repeated `--oauth-acl` entries. Confirm providers using the installed `--help`.
- Network edge: restrict inbound source IP/CIDR values using server `--allow-ips`.

Never confuse visitor/data-plane authentication with the shared tunnel token. Avoid embedding credentials in generated YAML or systemd output; use Kubernetes Secrets, environment files, or service credentials with appropriate permissions.

## Run persistently and verify

Use `--generate=systemd` or `--generate=k8s_yaml` where supported, inspect generated output, then supply secrets separately. Verify the upstream locally before debugging the tunnel. Check port 8123 reachability, DNS A/AAAA records, server/client logs, certificate issuance, and `inlets-pro status` in that order.
