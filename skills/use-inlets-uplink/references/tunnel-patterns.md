# Uplink tunnel patterns

Use these as templates; replace names, namespaces, ports, domains, and private targets. Every example assumes an `inlets-uplink-license` Secret in the tunnel namespace. Set the tenant once for CLI examples:

```bash
export TUNNEL_NS="tenant-name"
```

`TENANT_NAMESPACE` in YAML is an explicit placeholder; replace it with the selected tenant namespace before applying a manifest.

For HTTP routing, `upstreamDomains` contains short names. Uplink creates a Kubernetes Service for each entry, addressed as `<name>.<tunnel-namespace>`; use that address in client mappings and HTTP `Host` headers.

Before running a client, follow the secure connection workflow in [../SKILL.md](../SKILL.md) to set `$UPLINK_URL` and `$TOKEN_FILE` without displaying generated connection output or credentials. Execute preparation, client launch, verification, and cleanup in one shell invocation so temporary variables and traps remain active. Do not copy a URL path from these examples; router paths vary by deployment.

## Single TCP upstream

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: postgres
  namespace: TENANT_NAMESPACE
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: TENANT_NAMESPACE
  tcpPorts: [5432]
```

Equivalent creation and client connection:

```bash
inlets-pro tunnel create postgres -n "$TUNNEL_NS" --port 5432
inlets-pro uplink client \
  --url "$UPLINK_URL" \
  --token-file "$TOKEN_FILE" \
  --upstream 5432=postgres.internal:5432
```

## Multiple TCP upstreams

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: private-services
  namespace: TENANT_NAMESPACE
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: TENANT_NAMESPACE
  tcpPorts: [2222, 5432]
```

```bash
inlets-pro tunnel create private-services -n "$TUNNEL_NS" \
  --port 2222 --port 5432
inlets-pro uplink client \
  --url "$UPLINK_URL" \
  --token-file "$TOKEN_FILE" \
  --upstream 2222=ssh.internal:22 \
  --upstream 5432=postgres.internal:5432
```

## Single HTTP upstream

No `upstreamDomains` entry is required for the default HTTP service.

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: web
  namespace: TENANT_NAMESPACE
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: TENANT_NAMESPACE
```

```bash
inlets-pro tunnel create web -n "$TUNNEL_NS"
inlets-pro uplink client \
  --url "$UPLINK_URL" \
  --token-file "$TOKEN_FILE" \
  --upstream http://127.0.0.1:8080
```

The service is reachable inside the control-plane cluster through a generated Service. Never infer its name, DNS address, or port; inspect it with `kubectl get svc -n "$TUNNEL_NS"`.

## Multiple HTTP upstreams

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: platform
  namespace: TENANT_NAMESPACE
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: TENANT_NAMESPACE
  upstreamDomains:
  - gateway
  - prometheus
```

```bash
inlets-pro tunnel create platform -n "$TUNNEL_NS" \
  --upstream gateway --upstream prometheus
inlets-pro uplink client \
  --url "$UPLINK_URL" \
  --token-file "$TOKEN_FILE" \
  --upstream "gateway.${TUNNEL_NS}=http://gateway.openfaas:8080" \
  --upstream "prometheus.${TUNNEL_NS}=http://prometheus.monitoring:9090"
```

Inspect the generated Services and use their actual DNS and Host values. For example, after confirming the values:

```bash
curl -H "Host: gateway.${TUNNEL_NS}" "http://OBSERVED_SERVICE_ADDRESS:OBSERVED_PORT/"
curl -H "Host: prometheus.${TUNNEL_NS}" "http://OBSERVED_SERVICE_ADDRESS:OBSERVED_PORT/"
```

## Mixed TCP and HTTP

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: customer-edge
  namespace: TENANT_NAMESPACE
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: TENANT_NAMESPACE
  tcpPorts: [6443]
  upstreamDomains:
  - grafana
  - gateway
```

```bash
inlets-pro tunnel create customer-edge -n "$TUNNEL_NS" \
  --port 6443 \
  --upstream grafana \
  --upstream gateway

inlets-pro uplink client \
  --url "$UPLINK_URL" \
  --token-file "$TOKEN_FILE" \
  --upstream 6443=kubernetes.default.svc:443 \
  --upstream "grafana.${TUNNEL_NS}=http://grafana.monitoring:3000" \
  --upstream "gateway.${TUNNEL_NS}=http://gateway.openfaas:8080"
```

## Stable token reference

```bash
export TUNNEL_NAME="customer-edge"
export TOKEN_SECRET="${TUNNEL_NAME}-token"
umask 077
TOKEN_FILE="$(mktemp)"
trap 'rm -f "$TOKEN_FILE"' EXIT
openssl rand -base64 32 | tr -d '\n' > "$TOKEN_FILE"
kubectl create secret generic "$TOKEN_SECRET" \
  -n "$TUNNEL_NS" --from-file=token="$TOKEN_FILE"
```

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: customer-edge
  namespace: TENANT_NAMESPACE
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: TENANT_NAMESPACE
  tokenRef:
    name: customer-edge-token
    namespace: TENANT_NAMESPACE
```

Keep the Secret in the same namespace as the Tunnel.
