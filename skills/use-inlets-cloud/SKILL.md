---
name: use-inlets-cloud
description: Creates, connects, secures, and manages hosted Inlets Cloud HTTP and ingress tunnels with the inlets-pro cloud CLI. Use for CLI/dashboard onboarding, access-token login, generated tryinlets.dev or verified custom domains, HTTPS exposure, Bearer-protected HTTP upstreams, one-command directory sharing, snimux access to multiple SSH hosts or K3s APIs, systemd/launchd services, ACLs, or tunnel lifecycle operations.
---

# Use Inlets Cloud

Drive Inlets Cloud from the terminal with the `inlets-pro cloud` plugin. Use the web dashboard only for account/session tasks the CLI cannot perform, then tell the user the exact page and action required.

Docs: https://docs.inlets.dev/cloud/cli/
Dashboard: https://cloud.inlets.dev/

## Inspect and install the CLIs

Check the locally installed commands before assuming flags, because the independently versioned Cloud plugin can lag behind the current docs.

```bash
inlets-pro version
inlets-pro cloud version
inlets-pro cloud --help
```

If the plugin is missing or stale, install the latest release and inspect it again:

```bash
inlets-pro plugin get cloud
inlets-pro cloud version
```

## Hand off browser-only steps clearly

If the user has no account, tell them to register at `https://cloud.inlets.dev/register` using the email associated with their Inlets subscription. If they already have access:

1. Open `https://cloud.inlets.dev/` and sign in with the emailed magic link or verification code.
2. Open **CLI** in the sidebar for installation commands if `inlets-pro` or its Cloud plugin is missing.
3. Open **Access Tokens**, choose **Create New Token**, give it a recognizable name such as `laptop` or `ci`, select an appropriate expiry, and click **Create token**.
4. Copy the token immediately; the dashboard displays its value only once. Ask the user to save it to a password manager or a mode-0600 file, never paste it into chat.
5. Return to the terminal and log in. Prefer `--token-file` if the installed plugin exposes it:

```bash
chmod 600 ./inlets-cloud-token.txt
inlets-pro cloud auth login --token-file ./inlets-cloud-token.txt
inlets-pro cloud auth whoami
```

For an older plugin without `--token-file`, use `--token` only after warning that an inline value can be retained in shell history. Recommend updating the plugin first. The Cloud CLI sends this user access token as Bearer authentication to the management API and stores its login configuration locally. Use `inlets-pro cloud auth logout` to remove the active local login; revoke a compromised token from **Access Tokens** in the dashboard.

## Choose a region

Always list regions instead of relying on a default or a remembered list:

```bash
inlets-pro cloud region list
```

Use the returned `NAME` with `--region`. Check `HTTPS_GENERATED` and `HTTPS_CUSTOM` capabilities before choosing a region.

## Create an HTTP tunnel with a generated domain

HTTP is the default mode. With no custom domain, Inlets Cloud allocates an HTTPS `tryinlets.dev` hostname:

```bash
inlets-pro cloud tunnel create my-app --region cambs1
```

Use `--generated` when explicit intent is useful in automation; it is equivalent for HTTP mode:

```bash
inlets-pro cloud tunnel create my-app --generated --region cambs1
```

Read the assigned hostname from the create output, or query the tunnel/list output. Do not predict or hard-code the generated name.

## Create an HTTP tunnel with a custom domain

Register and verify the apex domain before creating a tunnel for one of its hostnames:

```bash
inlets-pro cloud domain add example.com
inlets-pro cloud domain get example.com
```

Tell the user to open their DNS provider and create the exact TXT challenge returned by `domain get` at the apex (`@` in many dashboards). Do not invent the value. After public DNS propagation:

```bash
inlets-pro cloud domain verify example.com
inlets-pro cloud tunnel create my-app \
  --domain app.example.com \
  --region cambs1
```

Use `domain list` to inspect registrations. Before `domain delete`, confirm no active tunnel depends on it.

## Connect an HTTP upstream

Have the Cloud CLI resolve the tunnel URL and token:

```bash
inlets-pro cloud tunnel connect my-app \
  --upstream http://127.0.0.1:3000
```

Run the printed `inlets-pro uplink client` command on the private network where the upstream is reachable. Treat its embedded tunnel token as a secret. Test the upstream locally before testing the public generated/custom HTTPS URL.

### Require Bearer authentication from visitors

Cloud management login and HTTP visitor authentication are separate. To protect the exposed HTTP service, first obtain the generated client command, then add `--bearer-token-file` to that `inlets-pro uplink client` command:

```bash
openssl rand -base64 32 | tr -d '\n' > http-bearer-token.txt
chmod 600 http-bearer-token.txt

# Preserve the --url, tunnel --token/--token-file, and --upstream values
# printed by: inlets-pro cloud tunnel connect my-app
inlets-pro uplink client \
  --url wss://CLOUD_URL \
  --token-file ./cloud-tunnel-token.txt \
  --upstream http://127.0.0.1:3000 \
  --bearer-token-file ./http-bearer-token.txt
```

If the generated command contains inline `--token`, preserve that value without displaying or committing it. `--bearer-token` is also available, but prefer the file variant. The two Bearer flags are mutually exclusive, require at least one HTTP upstream, and do not protect ingress/TCP pass-through traffic.

Verify the public endpoint rejects anonymous requests and accepts the header:

```bash
curl -i https://PUBLIC_DOMAIN/
curl -i -H "Authorization: Bearer $(cat http-bearer-token.txt)" \
  https://PUBLIC_DOMAIN/
```

## Share a directory with the fileserver shortcut

Use an HTTP tunnel. Obtain its generated client command with `cloud tunnel connect`, preserve the URL and tunnel token, remove its `--upstream` argument, and add `--fs`:

```bash
inlets-pro uplink client \
  --url wss://CLOUD_URL \
  --token-file ./cloud-tunnel-token.txt \
  --fs ./directory-to-share
```

`--fs` and `--upstream` cannot be combined. The client starts a loopback fileserver on an ephemeral port and prints a randomized `/share/<token>/` path. Share the complete public URL including that path. To keep the path stable across restarts, add `--fs-token` with a strong secret; otherwise allow the client to generate it. Add `--bearer-token-file` for an additional required Authorization header when the installed version supports it.

## Create an ingress tunnel

Ingress mode passes TCP ports 80 and 443 through to a local reverse proxy, Kubernetes Ingress Controller, or similar component. It does not terminate HTTPS in Inlets Cloud and requires at least one verified custom domain.

```bash
inlets-pro cloud tunnel create ingress edge \
  --domain app.example.com \
  --region cambs1

inlets-pro cloud tunnel connect edge \
  --upstream 80=ingress-nginx-controller.ingress-nginx:80 \
  --upstream 443=ingress-nginx-controller.ingress-nginx:443
```

Attach or detach additional verified domains with `cloud tunnel domain add|remove`. Configure TLS and HTTP authentication at the tunneled reverse proxy, not with the Uplink HTTP Bearer option.

## Route SSH and K3s with snimux

Use `snimux` to route several SSH hosts and K3s APIs through one Inlets Cloud ingress tunnel on public port 443. Read [references/snimux.md](references/snimux.md) before configuring this workflow. It covers:

- creating and extending the Cloud ingress tunnel and its domains;
- mapping public port 443 to the private snimux listener on 8443;
- configuring multiple SSH upstreams and K3s TLS passthrough;
- safely updating local OpenSSH and kubeconfig files;
- running both required processes with systemd or macOS launchd;
- checking logs and disabling, unloading, or removing each service.

Apply these rules before creating or changing anything:

- **Use only an ingress tunnel with a verified custom domain.** An HTTP tunnel and a generated `tryinlets.dev` domain cannot carry snimux traffic. Never create either as a snimux experiment or fallback.
- Run `inlets-pro cloud domain list` and `inlets-pro cloud tunnel list --verbose` first. Reuse a verified domain or suitable ingress tunnel only when the user authorized it. If no verified custom domain is available, stop and ask the user to register or select one.
- Map public ingress port `443` to the private snimux listener, normally `127.0.0.1:8443`. When Uplink and snimux run on the same host, keep this listener on loopback; do not add `--data-addr '0.0.0.0:'`.
- Treat snimux and Uplink as two separate long-running processes. Start and verify both on the private host; do not run either on the operator's workstation unless that placement was requested.
- Do not introduce HTTP upstream URLs, stunnel, nginx, TLS termination, WebSocket bridges, or alternate ports to repair this workflow. Diagnose the prescribed raw port mapping instead.
- Do not claim success until an actual SSH or K3s request succeeds. For SSH, finish by giving the user an exact `ssh` command containing the snimux `ProxyCommand`.

Use `inlets-pro snimux server` and `inlets-pro snimux connect`. Inspect `inlets-pro snimux --help` before acting.

## Generate persistent clients

Use Cloud output rather than hand-writing tunnel credentials:

```bash
inlets-pro cloud tunnel connect my-app --format k8s \
  --namespace default --upstream http://my-service:8080 > inlets-client.yaml

inlets-pro cloud tunnel connect my-app --format systemd \
  --upstream http://127.0.0.1:8080 > inlets-client.service
```

Inspect generated output for inline secrets before saving or committing it. `--format k8s` is the current Cloud plugin spelling; do not substitute Uplink plugin formats without checking `cloud tunnel connect --help`.

## Inspect, restrict, and remove

```bash
inlets-pro cloud tunnel list
inlets-pro cloud tunnel list --verbose
inlets-pro cloud acl list my-app
inlets-pro cloud acl set my-app \
  --allow 203.0.113.0/24 --allow 198.51.100.5
```

`acl set` replaces the whole allow list. Confirm the full intended list so the user does not lock themselves out. Enable Proxy Protocol on ingress tunnels only when the local receiver is configured for the same version.

Before deletion, resolve a name/domain to the exact tunnel, show it to the user, and then delete explicitly:

```bash
inlets-pro cloud tunnel delete my-app
```

Use `--json` where exposed for scripting and avoid parsing human-readable tables.
