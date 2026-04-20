---
name: use-inletsctl
description: Creates and manages inlets tunnel exit-servers on cloud VMs using inletsctl. Use when asked to provision, create, or delete tunnel servers on cloud providers like DigitalOcean, AWS EC2, GCE, Azure, Linode, Hetzner, Scaleway, OVH, or Vultr.
---

# Using inletsctl

Automate provisioning of inlets exit-servers (tunnel servers) on cloud infrastructure.

Docs: https://docs.inlets.dev/reference/inletsctl/
Source: https://github.com/inlets/inletsctl

## Download inlets-pro binary

```bash
inletsctl download

# Specific version
inletsctl download --version 0.9.40
```

## Creating Exit-Servers

Either `--letsencrypt-domain` (for HTTPS) or `--tcp` (for TCP) must be specified.

### HTTPS tunnel with Let's Encrypt

Terminates TLS at the exit-server with automatic Let's Encrypt certificates:

```bash
inletsctl create my-tunnel \
  --provider digitalocean \
  --region lon1 \
  --access-token-file $HOME/do-access-token \
  --letsencrypt-domain app.example.com \
  --letsencrypt-issuer prod
```

Multiple domains on the same tunnel:

```bash
inletsctl create my-tunnel \
  --letsencrypt-domain app1.example.com \
  --letsencrypt-domain app2.example.com
```

### TCP tunnel

For raw TCP traffic (SSH, databases, etc.):

```bash
inletsctl create my-tcp-tunnel \
  --tcp \
  --provider digitalocean \
  --region lon1 \
  --access-token-file $HOME/do-access-token
```

If no name argument is given, a random name is generated.

After creation, inletsctl prints a connection command with the server IP and auth token. Run that command locally, updating `--upstream` and `--ports` as needed.

## Deleting Exit-Servers

Delete by ID or IP (both are printed at creation time):

```bash
# By ID
inletsctl delete --provider digitalocean \
  --access-token-file $HOME/do-access-token \
  --id 164857028

# By IP
inletsctl delete --provider digitalocean \
  --access-token-file $HOME/do-access-token \
  --ip 209.97.131.180
```

## Forwarding Kubernetes Services Locally

Forward a Kubernetes service to your local machine:

```bash
# HTTP service
inletsctl kfwd --from my-service:8080 --if 192.168.0.14

# TCP service
inletsctl kfwd --from nats:4222 --if 192.168.0.14 --tcp
```

`--if` must be an IP on your machine reachable from the cluster. Use `--namespace` for non-default namespaces.

## Supported Providers and Defaults

| Provider | `--provider` | Default Region | Default Plan | Default OS |
|---|---|---|---|---|
| DigitalOcean | `digitalocean` | `lon1` | `s-1vcpu-512mb-10gb` | `debian-13-x64` |
| AWS EC2 | `ec2` | `eu-west-1` | `t3.nano` | Ubuntu 22.04 amd64 |
| Google Compute Engine | `gce` | (required) | `f1-micro` | Ubuntu Minimal 22.04 |
| Azure | `azure` | (required) | `Standard_B1ls` | Ubuntu 22.04 LTS Gen2 |
| Linode | `linode` | `eu-west` | `g6-nanode-1` | `linode/ubuntu22.04` |
| Hetzner | `hetzner` | `hel1` | `cx23` | `debian-13` |
| Scaleway | `scaleway` | `fr-par-1` | `DEV1-S` | `ubuntu-jammy` |
| OVHcloud | `ovh` | `DE1` | `s1-2` | Ubuntu 22.04 |
| Vultr | `vultr` | `LHR` | `201` (1GB/25GB) | Ubuntu 22.04 x64 |

## Provider-Specific Configuration

### DigitalOcean

```bash
inletsctl create my-tunnel \
  --provider digitalocean \
  --region lon1 \
  --access-token-file $HOME/do-access-token \
  --tcp
```

### AWS EC2

Required: `--access-token-file` (access key) and `--secret-key-file` (secret key).

```bash
inletsctl create my-tunnel \
  --provider ec2 \
  --region eu-west-1 \
  --access-token-file ./access-key.txt \
  --secret-key-file ./secret-key.txt \
  --tcp
```

With temporary credentials (STS), add `--session-token-file ./session-token.txt`.

Optional: `--vpc-id` and `--subnet-id` (both must be set together), `--aws-key-name`.

### Google Compute Engine (GCE)

Required: `--project-id` and `--region` (no default region). `--access-token-file` takes a service account key JSON.

```bash
inletsctl create my-tunnel \
  --provider gce \
  --project-id $PROJECTID \
  --region us-central1 \
  --access-token-file key.json \
  --zone us-central1-a \
  --tcp
```

Default zone: `us-central1-a`. GCE VMs get ephemeral IPs — reserve a static IP for a stable tunnel address.

### Azure

Required: `--subscription-id`. `--access-token-file` takes a service principal JSON (`az ad sp create-for-rbac --sdk-auth`).

```bash
inletsctl create my-tunnel \
  --provider azure \
  --region eastus \
  --subscription-id $SUBSCRIPTION_ID \
  --access-token-file ~/client_credentials.json \
  --tcp
```

### Linode

```bash
inletsctl create my-tunnel \
  --provider linode \
  --region us-east \
  --access-token $TOKEN \
  --tcp
```

### Hetzner

Regions: `hel1` (Helsinki, default), `nur1` (Nuremberg), `fsn1` (Falkenstein).

```bash
inletsctl create my-tunnel \
  --provider hetzner \
  --region hel1 \
  --access-token $TOKEN \
  --tcp
```

### Scaleway

Required: `--secret-key` and `--organisation-id`. Region fixed to `fr-par-1`.

```bash
inletsctl create my-tunnel \
  --provider scaleway \
  --access-token $TOKEN \
  --secret-key $SECRET_KEY \
  --organisation-id $ORG_ID \
  --tcp
```

### OVHcloud

Required: `--secret-key`, `--consumer-key`, and `--project-id`.

Endpoints: `ovh-eu` (default), `ovh-us`, `ovh-ca`, `soyoustart-eu`, `soyoustart-ca`, `kimsufi-eu`, `kimsufi-ca`.

```bash
inletsctl create my-tunnel \
  --provider ovh \
  --access-token $APPLICATION_KEY \
  --secret-key $APPLICATION_SECRET \
  --consumer-key $CONSUMER_KEY \
  --project-id $PROJECT_ID \
  --endpoint ovh-eu \
  --tcp
```

### Vultr

Default region: `LHR` (London).

```bash
inletsctl create my-tunnel \
  --provider vultr \
  --region LHR \
  --access-token $TOKEN \
  --tcp
```

## Key Flags Reference

| Flag | Description |
|---|---|
| `--provider` / `-p` | Cloud provider (default: `digitalocean`) |
| `--region` / `-r` | Cloud region (provider-specific default) |
| `--plan` / `-s` | Override default VM plan/size |
| `--zone` / `-z` | Zone (GCE only, default: `us-central1-a`) |
| `--access-token` / `-a` | Access token inline |
| `--access-token-file` / `-f` | Read access token from file |
| `--tcp` | Create a TCP tunnel server |
| `--letsencrypt-domain` | Domain for Let's Encrypt cert (repeatable) |
| `--letsencrypt-issuer` | `prod` (default) or `staging` |
| `--inlets-token` / `-t` | Custom auth token (auto-generated if omitted) |
| `--inlets-version` | inlets binary version (default: `0.11.8`) |
| `--poll` / `-n` | Poll interval for provisioning status (default: `2s`) |
