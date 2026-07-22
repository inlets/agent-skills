---
name: use-inlets-operator
description: Installs, inspects, and operates the inlets-operator lifecycle for Kubernetes LoadBalancer Services and Tunnel CRs. Use when exposing a private cluster Service with a cloud VM/public IP, checking operator resources with kubectl, recreating a tunnel, or safely deleting operator-managed infrastructure.
---

# Use inlets-operator

Keep the workflow centered on Kubernetes resources. The operator watches eligible `Service` objects of type `LoadBalancer`, provisions a cloud VM running an Inlets Pro server, runs the client in-cluster, creates a `Tunnel` CR (`tunnels.operator.inlets.dev`), and writes the public IP to the Service.

Docs: https://docs.inlets.dev/reference/inlets-operator/

## Install

Collect the cloud provider, region, access credential requirements, and Inlets Pro license before installing. Never guess a cloud region; ask the user or use an explicitly configured value. Prefer the user's existing Helm or arkade workflow and inspect current flags:

```bash
arkade install inlets-operator --help
helm show values inlets/inlets-operator
```

Before installing, inspect existing LoadBalancer Services and controllers. K3s ServiceLB, MetalLB, or another controller may already handle `LoadBalancer` Services:

```bash
kubectl get svc -A
kubectl get pods -A
helm list -A
```

If another implementation is present, install with `annotatedOnly: true`; otherwise the operator may manage an existing Service unintentionally.

Keep the operator in its own namespace, commonly `inlets`. Pass credential file paths directly to installers; never read, decode, preview, or print their contents. With the default chart mounts, create the prerequisite Secrets using these exact data keys:

```bash
kubectl create namespace inlets
kubectl create secret generic inlets-access-key -n inlets \
  --from-file=inlets-access-key=/path/to/provider-token
kubectl create secret generic inlets-license -n inlets \
  --from-file=license=/path/to/inlets-license
```

Inspect only Secret names and data-key names when diagnosing mounts. Do not copy a base64 value from an existing Secret into `--from-literal`; that can double-encode the credential. Store provider credentials and the license only in Secrets, never in committed manifests, agent output, or process arguments.

Do not use installer `--wait` until the namespace and required Secrets exist. After installation, verify the actual chart values and Deployment rather than relying only on installer output:

```bash
helm get values inlets-operator -n inlets
kubectl rollout status deployment/inlets-operator -n inlets --timeout=2m
kubectl get pods -n inlets
```

## Expose a Service

Create or patch the intended Service to `type: LoadBalancer`, then observe rather than manually creating the operator's Tunnel CR:

```bash
kubectl get svc -A
kubectl get tunnels.operator.inlets.dev -A
kubectl get pods -n inlets
kubectl get events -A --sort-by=.lastTimestamp
```

The Tunnel's `HOSTSTATUS` and `HOSTIP` show provisioning state; the same public IP should appear under the Service's `EXTERNAL-IP`.

If another LoadBalancer implementation is installed, configure the chart with `annotatedOnly: true` and annotate only Internet-facing Services:

```bash
kubectl annotate service my-service operator.inlets.dev/manage=1 --overwrite
```

Confirm the exact selector/annotation behavior against the installed chart values before changing it.

Inspect labels before using selectors; do not invent a Pod label from a resource name. Treat operator and client logs as potentially sensitive because they may contain customer identity, license identity, host IDs, or public IPs. Read only the relevant lines and redact personal information when reporting results.

Use time-bounded checks for provisioning and data-path tests. A client may initially fail while the cloud VM boots; correlate Service state, Tunnel `HOSTSTATUS`, operator events, and client readiness before declaring failure.

## Understand lifecycle

- The `Service` owns the desired lifecycle. Deleting it permanently removes the associated tunnel and cloud VM.
- Deleting only the generated `Tunnel` CR requests recreation; the operator notices the still-present Service and provisions it again.
- Before deleting the Kubernetes cluster or uninstalling the operator, delete managed LoadBalancer Services and wait until their Tunnel CRs and cloud VMs are gone.
- If the cluster was deleted first, the operator cannot clean up. Find and remove the orphaned VM through the cloud provider; its name normally matches the Service.

Use explicit namespace and resource names for every delete. Before destructive operations, show the Service, Tunnel, and public IP/cloud identity being targeted. A final cleanup request means delete the Service and do not recreate it; recreation is only appropriate when the user explicitly asks to replace the Tunnel while retaining the Service.

```bash
kubectl get svc my-service -n default -o wide
kubectl get tunnels.operator.inlets.dev my-service-tunnel -n default -o wide
kubectl delete svc my-service -n default
kubectl wait --for=delete tunnels.operator.inlets.dev/my-service-tunnel \
  -n default --timeout=10m
```

Verify actual state after cleanup: the Service, generated client workload, and Tunnel CR must be absent before uninstalling the operator. Confirm provider deletion through operator events/logs and, when required, the provider API or CLI. Pass provider authentication through a supported credential file, restrictive config/header file, or secret store; never expand a token into command arguments.

For recreation only:

```bash
kubectl delete tunnels.operator.inlets.dev my-service-tunnel -n default
kubectl get tunnels.operator.inlets.dev,svc -n default -w
```

Before reporting completion or uninstall status, query the cluster again; do not rely on an earlier command or inferred state. Troubleshoot in this order: Service eligibility/annotation, chart values, operator events, provider Secret key names, provider quota/region availability, license Secret key name, operator/client Pods, then public port reachability.
