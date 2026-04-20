---
name: use-inlets-cloud
description: Creates and manages inlets cloud tunnels (HTTP and ingress) using the inlets-pro cloud CLI. Use when asked to expose a service, create a tunnel, or manage tunnels with inlets cloud.
---

# Using Inlets Cloud

Manage tunnels on Inlets Cloud using `inlets-pro cloud`.

Docs: https://docs.inlets.dev/cloud/

## Authentication

Log in before any other command:

```bash
inlets-pro cloud auth login --token TOKEN
```

Check current user:

```bash
inlets-pro cloud auth whoami
```

## Regions

List available regions:

```bash
inlets-pro cloud region list
```

The `--region` flag defaults to `eu-west-1`. Run `region list` to see current options.

## Custom Domains

Before using a custom domain with an HTTP tunnel, register and verify it:

```bash
# Add an apex domain
inlets-pro cloud domain add example.com

# Get verification challenge (TXT record)
inlets-pro cloud domain get example.com

# Verify after creating the TXT record
inlets-pro cloud domain verify example.com
```

List registered domains:

```bash
inlets-pro cloud domain list
```

## Creating Tunnels

### HTTP tunnel with generated domain (tryinlets.dev)

Quick way to get a HTTPS URL with a random subdomain:

```bash
inlets-pro cloud tunnel create my-tunnel --generated --region cambs1
```

### HTTP tunnel with custom domain

Expose a service on your own domain (domain must be verified first):

```bash
inlets-pro cloud tunnel create my-tunnel --domain app.example.com --region cambs1
```

### Ingress tunnel

Expose ports 80 and 443 with TCP pass-through to a local reverse proxy or Kubernetes Ingress Controller:

```bash
inlets-pro cloud tunnel create ingress my-tunnel --domain app.example.com --region cambs1
```

Multiple domains can be attached:

```bash
inlets-pro cloud tunnel create ingress my-tunnel \
  --domain app.example.com \
  --domain api.example.com \
  --region cambs1
```

## Connecting to a Tunnel

Get the connection command for a tunnel:

```bash
# CLI connection string
inlets-pro cloud tunnel connect my-tunnel

# Kubernetes YAML
inlets-pro cloud tunnel connect my-tunnel --format k8s --upstream UPSTREAM

# systemd unit
inlets-pro cloud tunnel connect my-tunnel --format systemd --upstream UPSTREAM
```

The `--upstream` flag specifies the local target, e.g. `--upstream http://127.0.0.1:8080`.

For Kubernetes format, `--namespace` and `--inlets-version` can also be set.

## Listing and Deleting Tunnels

```bash
# List all tunnels
inlets-pro cloud tunnel list

# List with RX/TX metrics (slower)
inlets-pro cloud tunnel list -v

# Delete a tunnel by name or domain
inlets-pro cloud tunnel delete my-tunnel
```

## Managing Ingress Tunnel Domains

Attach or detach domains on an existing ingress tunnel:

```bash
# Add a domain
inlets-pro cloud tunnel domain add my-tunnel blog.example.com

# Remove a domain
inlets-pro cloud tunnel domain remove my-tunnel blog.example.com
```

## Proxy Protocol (Ingress Tunnels)

Enable proxy protocol to preserve client IP addresses:

```bash
# Set proxy protocol (none, v1, v2)
inlets-pro cloud tunnel proxy-protocol set my-tunnel v2
```

The tunneled service must be configured to accept the proxy protocol header.

## ACLs (IP Filtering)

Restrict access to a tunnel by IP/CIDR:

```bash
# List current ACL entries
inlets-pro cloud acl list my-tunnel

# Set allowed IPs (replaces existing entries)
inlets-pro cloud acl set my-tunnel --allow 203.0.113.0/24 --allow 198.51.100.5
```

## Common Workflows

### Expose a local web app with HTTPS

```bash
inlets-pro cloud tunnel create my-app --generated --region cambs1
inlets-pro cloud tunnel connect my-app --upstream http://127.0.0.1:3000
# Run the printed inlets-pro uplink client command
```

### Expose a Kubernetes Ingress Controller

```bash
inlets-pro cloud domain add example.com
inlets-pro cloud domain verify example.com
inlets-pro cloud tunnel create ingress k8s-ingress --domain app.example.com --region cambs1
inlets-pro cloud tunnel connect k8s-ingress --format k8s --upstream http://ingress-nginx-controller.ingress-nginx:80
# Apply the generated YAML to your cluster
```
