# Route SSH and K3s with snimux

Use this workflow to expose several private SSH servers and K3s API servers through a single Inlets Cloud ingress tunnel. `snimux` wraps SSH in TLS so a hostname can be carried as SNI; K3s already speaks TLS, so its connection uses passthrough mode.

## Contents

- [Plan names and endpoints](#plan-names-and-endpoints)
- [Create the Cloud ingress tunnel](#create-the-cloud-ingress-tunnel)
- [Configure snimux upstreams](#configure-snimux-upstreams)
- [Connect the Cloud tunnel](#connect-the-cloud-tunnel)
- [Update local SSH configuration](#update-local-ssh-configuration)
- [Expose a K3s API](#expose-a-k3s-api)
- [Run persistently with systemd](#run-persistently-with-systemd)
- [Run persistently with macOS launchd](#run-persistently-with-macos-launchd)
- [Troubleshoot and remove](#troubleshoot-and-remove)

## Plan names and endpoints

Collect these values before changing anything:

| Placeholder | Meaning | Example |
|---|---|---|
| `SSH_DOMAIN` | Verified apex domain | `example.com` |
| `SSH_HOSTS` | Public names used as SNI routes | `nuc.example.com`, `nas.example.com` |
| `K3S_HOST` | Optional public/SNI name for Kubernetes | `k3s.example.com` |
| `CLOUD_EDGE` | Public Inlets Cloud data-plane hostname from the tunnel details/connect output | `cambs1.uplink.inlets.dev` |
| `CLOUD_PORT` | Public ingress data-plane port | `443` |
| `SNIMUX_ADDR` | Private listener reached by the Uplink client | `127.0.0.1:8443` |

Do not guess `CLOUD_EDGE`; take it from the created tunnel's connection details. Ensure every hostname is attached to the same ingress tunnel and has the DNS record instructed by Inlets Cloud. The public hostname is also the snimux route key, so spelling must match across Cloud, DNS, `snimux.yaml`, SSH config, and kubeconfig.

Harden every SSH server before exposure: disable root and password login, require keys, keep host keys verified, and expose only intended hosts. Prefer Cloud ACLs and snimux per-upstream `allow` lists where client source addresses are stable.

## Create the Cloud ingress tunnel

Register and verify the apex domain as described in the main skill. Start with one SSH host, then add the others:

```bash
inlets-pro cloud tunnel create ingress remote-access \
  --domain nuc.example.com \
  --region cambs1

inlets-pro cloud tunnel domain add remote-access nas.example.com
inlets-pro cloud tunnel domain add remote-access k3s.example.com
```

Follow the Cloud output/dashboard instructions for DNS. Typically each hostname is a CNAME to the region's ingress endpoint, but use the exact target displayed by Inlets Cloud. Verify before proceeding:

```bash
dig +short nuc.example.com
dig +short nas.example.com
dig +short k3s.example.com
inlets-pro cloud tunnel list
```

If the tunnel receives Proxy Protocol, set the same supported version on the Cloud tunnel and snimux server. Otherwise leave it disabled on both sides. A mismatch prevents correct connections or source-address ACL evaluation.

## Configure snimux upstreams

Create a YAML file on the private gateway host that can reach every backend:

```yaml
upstreams:
- name: nuc.example.com
  upstream: 192.168.1.20:22
- name: nas.example.com
  upstream: 192.168.1.30:2222
- name: k3s.example.com
  upstream: 192.168.1.40:6443
  passthrough: true
```

Set every `upstream` to `host:port`. Use `passthrough: true` for K3s endpoints so snimux routes by SNI without terminating Kubernetes TLS. Do not set passthrough for SSH, because snimux must unwrap the TLS envelope added by `snimux connect`.

Optionally restrict a route by source IP/CIDR:

```yaml
- name: nuc.example.com
  upstream: 192.168.1.20:22
  allow:
  - 198.51.100.25
  - 203.0.113.0/24
```

Source ACLs require the real source address. If Inlets Cloud sends Proxy Protocol, enable the matching `--proxy-protocol v1` or `v2` mode on snimux. Confirm the currently supported values with `inlets-pro snimux server --help` and the Cloud tunnel settings.

Validate routing locally before making it persistent:

```bash
inlets-pro snimux server --port 8443 ./snimux.yaml
```

The default bind address is loopback (`127.0.0.1:`), which is appropriate when the Uplink client runs on the same machine. Use `--data-addr '0.0.0.0:'` only when the client is on another trusted machine and the network path is restricted.

## Connect the Cloud tunnel

Ask the Cloud CLI to print the ingress client command:

```bash
inlets-pro cloud tunnel connect remote-access \
  --upstream 443=127.0.0.1:8443
```

Run the resulting `inlets-pro uplink client` on the same private host as snimux. Preserve the generated `--url` and tunnel token, but ensure the upstream mapping is public port 443 to private snimux port 8443:

```bash
inlets-pro uplink client \
  --url wss://CLOUD_CONTROL_URL \
  --token-file /etc/inlets/remote-access.token \
  --upstream 443=127.0.0.1:8443
```

The Cloud Uplink client and snimux server are separate long-running processes; both must be healthy. Keep the tunnel token out of shell history and set its file mode to `0600`.

## Update local SSH configuration

Inspect `~/.ssh/config` first. Preserve existing content and avoid a broad wildcard that captures unrelated hosts. Prefer a separate included file so the Inlets block is easy to test and remove:

```sshconfig
# Add once near the top of ~/.ssh/config
Include ~/.ssh/config.d/inlets-cloud
```

Then create `~/.ssh/config.d/inlets-cloud` with explicit hosts or a suitably narrow suffix:

```sshconfig
Host nuc.example.com nas.example.com
    HostName %h
    Port 443
    User operator
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ProxyCommand /absolute/path/to/inlets-pro snimux connect CLOUD_EDGE:%p %h
```

Resolve `/absolute/path/to/inlets-pro` with `command -v inlets-pro`; an absolute path avoids GUI/launch-agent PATH differences. Replace `CLOUD_EDGE` with the data-plane hostname from Inlets Cloud. The destination `%h` becomes the SNI route; `%p` expands to 443.

Before editing, make a timestamped backup when the file already exists. Keep `~/.ssh` at mode `0700` and both config files at `0600`. Validate without connecting:

```bash
ssh -G nuc.example.com | grep -E '^(hostname|port|user|proxycommand) '
```

Then connect and confirm the host key fingerprint through a trusted channel on first use:

```bash
ssh -v nuc.example.com
scp ./file.txt nas.example.com:~/file.txt
```

To add a host, first attach its domain to the Cloud tunnel and configure DNS, then add an exactly matching `name`/`upstream` to `snimux.yaml`, restart snimux, and finally add it to the SSH Host list. To remove one, reverse those steps after confirming it is no longer needed.

## Expose a K3s API

K3s must present a certificate containing the same SNI name used in `snimux.yaml`. For a new installation with k3sup, include:

```bash
k3sup install --tls-san k3s.example.com ...
```

For an existing K3s server, update its service configuration to add `--tls-san k3s.example.com`, then reload and restart K3s according to how it was installed. On a typical systemd installation:

```bash
sudo systemctl daemon-reload
sudo systemctl restart k3s
sudo journalctl -u k3s -n 100 --no-pager
```

Do not patch an existing K3s unit blindly: inspect `systemctl cat k3s` and its environment/config files first, preserve all existing arguments, and confirm the certificate is reissued with the new SAN.

Copy the kubeconfig to the client securely and edit only the intended cluster entry. Preserve its certificate-authority data and credentials. Point it at the public name on port 443 and set the TLS server name to the snimux route:

```yaml
clusters:
- cluster:
    certificate-authority-data: EXISTING_VALUE
    server: https://k3s.example.com:443
    tls-server-name: k3s.example.com
  name: home-k3s
```

The `snimux.yaml` entry must be `k3s.example.com`, point to `host:6443`, and set `passthrough: true`. Test the specific context without overwriting the user's current context:

```bash
kubectl --context home-k3s get --raw=/readyz
kubectl --context home-k3s get nodes
```

Repeat the pattern for additional clusters only when sharing this SSH/K3s tunnel is useful: give each cluster a unique public/SNI hostname, add that SAN to the corresponding K3s server certificate, add a distinct passthrough route, attach the domain to the Cloud tunnel, and create a separate kubeconfig cluster entry. Never reuse one route name for different clusters. For ordinary HTTPS applications, prefer dedicated Inlets Cloud HTTP or ingress tunnels instead of routing them through snimux.

## Run persistently with systemd

Use separate services so each process has clear logs and restart behavior. Install the binary at a stable absolute path and place configuration/token files under `/etc/inlets` with restrictive ownership. Run them as a dedicated unprivileged `inlets` service account after granting it read access to only the required configuration and token files.

Create `/etc/systemd/system/inlets-cloud-snimux.service`:

```systemd
[Unit]
Description=Inlets snimux router
After=network-online.target
Wants=network-online.target

[Service]
User=inlets
Group=inlets
ExecStart=/usr/local/bin/inlets-pro snimux server --port 8443 /etc/inlets/snimux.yaml
Restart=always
RestartSec=5
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

Generate the Uplink client unit when supported, then inspect and save it as `/etc/systemd/system/inlets-cloud-remote-access.service`:

```bash
inlets-pro cloud tunnel connect remote-access \
  --format systemd \
  --upstream 443=127.0.0.1:8443
```

Prefer a `--token-file` in the installed unit over an inline token, and configure the generated unit to use the same unprivileged account after checking file access. After both unit files are in place:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now inlets-cloud-snimux.service
sudo systemctl enable --now inlets-cloud-remote-access.service
sudo systemctl status inlets-cloud-snimux.service inlets-cloud-remote-access.service
sudo journalctl -u inlets-cloud-snimux.service -f
sudo journalctl -u inlets-cloud-remote-access.service -f
```

Stop and disable without deleting files:

```bash
sudo systemctl disable --now inlets-cloud-snimux.service
sudo systemctl disable --now inlets-cloud-remote-access.service
```

Remove only after confirming the exact unit and config paths:

```bash
sudo systemctl disable --now inlets-cloud-snimux.service inlets-cloud-remote-access.service
sudo rm /etc/systemd/system/inlets-cloud-snimux.service
sudo rm /etc/systemd/system/inlets-cloud-remote-access.service
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

Treat removal of `/etc/inlets/snimux.yaml` and the tunnel token as a separate decision; keeping them permits recovery. Deleting the Inlets Cloud tunnel is also separate and may affect DNS and every configured host.

## Run persistently with macOS launchd

Run snimux and the Uplink client as separate per-user LaunchAgents. Resolve absolute paths first:

```bash
command -v inlets-pro
id -u
```

In plist files, do not use `~`, `$HOME`, or rely on shell expansion; substitute absolute `/Users/NAME/...` paths. Create `~/Library/LaunchAgents/dev.inlets.snimux.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>dev.inlets.snimux</string>
  <key>ProgramArguments</key>
  <array>
    <string>/ABSOLUTE/PATH/inlets-pro</string>
    <string>snimux</string><string>server</string>
    <string>--port</string><string>8443</string>
    <string>/Users/NAME/.config/inlets/snimux.yaml</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/Users/NAME/Library/Logs/inlets-snimux.log</string>
  <key>StandardErrorPath</key><string>/Users/NAME/Library/Logs/inlets-snimux.error.log</string>
</dict>
</plist>
```

Create `~/Library/LaunchAgents/dev.inlets.cloud-remote-access.plist` for the tunnel client:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>dev.inlets.cloud-remote-access</string>
  <key>ProgramArguments</key>
  <array>
    <string>/ABSOLUTE/PATH/inlets-pro</string>
    <string>uplink</string><string>client</string>
    <string>--url</string><string>wss://CLOUD_CONTROL_URL</string>
    <string>--token-file</string><string>/Users/NAME/.config/inlets/remote-access.token</string>
    <string>--upstream</string><string>443=127.0.0.1:8443</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/Users/NAME/Library/Logs/inlets-cloud-remote-access.log</string>
  <key>StandardErrorPath</key><string>/Users/NAME/Library/Logs/inlets-cloud-remote-access.error.log</string>
</dict>
</plist>
```

Validate and load them:

```bash
plutil -lint ~/Library/LaunchAgents/dev.inlets.snimux.plist
plutil -lint ~/Library/LaunchAgents/dev.inlets.cloud-remote-access.plist
launchctl bootstrap "gui/$(id -u)" ~/Library/LaunchAgents/dev.inlets.snimux.plist
launchctl bootstrap "gui/$(id -u)" ~/Library/LaunchAgents/dev.inlets.cloud-remote-access.plist
launchctl print "gui/$(id -u)/dev.inlets.snimux"
launchctl print "gui/$(id -u)/dev.inlets.cloud-remote-access"
tail -f ~/Library/Logs/inlets-snimux.error.log
tail -f ~/Library/Logs/inlets-cloud-remote-access.error.log
```

Restart after configuration changes:

```bash
launchctl kickstart -k "gui/$(id -u)/dev.inlets.snimux"
launchctl kickstart -k "gui/$(id -u)/dev.inlets.cloud-remote-access"
```

Unload without deleting the plists:

```bash
launchctl bootout "gui/$(id -u)" ~/Library/LaunchAgents/dev.inlets.snimux.plist
launchctl bootout "gui/$(id -u)" ~/Library/LaunchAgents/dev.inlets.cloud-remote-access.plist
```

To persistently disable labels, use `launchctl disable gui/$(id -u)/LABEL`; re-enable with `launchctl enable gui/$(id -u)/LABEL`. To remove them, boot them out first, confirm both exact paths, then delete only those two plist files. Keep the YAML/token files unless the user explicitly wants credentials and configuration removed.

Use `/Library/LaunchDaemons` and the `system` launchctl domain only when the services must run before user login. LaunchDaemons require root-owned plists, must not reference user-home secrets casually, and need separate ownership and least-privilege planning.

## Troubleshoot and remove

Check the path one hop at a time:

1. Confirm each private backend is reachable from the snimux host (`nc -vz HOST PORT`).
2. Confirm snimux listens on loopback port 8443.
3. Confirm the Uplink client reports a connection and maps `443=127.0.0.1:8443`.
4. Confirm Cloud lists the tunnel as connected and every domain is attached.
5. Confirm public DNS resolves to the target instructed by Cloud.
6. For SSH, inspect `ssh -G` and run `ssh -vvv HOST`.
7. For K3s, verify the certificate SAN, `passthrough: true`, kubeconfig `server`, and `tls-server-name` all use the same hostname.

Common failures:

- `No upstream for`: the SNI name does not exactly match a `name` in `snimux.yaml`.
- SSH host-key warning: stop and verify the backend fingerprint; never suppress `StrictHostKeyChecking` globally.
- TLS hostname error from kubectl: K3s lacks the chosen `--tls-san`, or kubeconfig `tls-server-name` differs.
- Immediate timeout: wrong `CLOUD_EDGE`, port 443 blocked, Cloud tunnel disconnected, or `dial_timeout` too short. Set `dial_timeout=5s` for `snimux connect` only when latency justifies it.
- ACL denies all clients: Proxy Protocol is missing/mismatched or the observed public source address is not allowed.

Removing one SSH host does not require deleting the tunnel: remove it from local SSH config, remove/restart the snimux route, detach the Cloud tunnel domain, and remove DNS after confirming no other user depends on it. Remove K3s access similarly, but leave its certificate SAN unless there is a concrete reason to rotate certificates.
