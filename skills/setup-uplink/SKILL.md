---
name: setup-uplink
description: Installs, configures, verifies, and upgrades self-hosted Inlets Uplink on an existing Kubernetes cluster using its Helm chart. Use for Uplink licensing, Traefik or other Kubernetes ingress controllers, Istio, TLS and DNS, tenant namespaces, and private tunnels or optional public per-tunnel and wildcard HTTPS exposure.
---

# Set Up Inlets Uplink

Install the Uplink management components in the user's existing cluster, then verify a connected tunnel with the selected visibility. This is the self-hosted Uplink product, distinct from hosted Inlets Cloud and standalone Inlets Pro servers.

Start with the current [installation documentation](https://docs.inlets.dev/uplink/installation/). Inspect the selected chart's values and rendered manifests when examples differ from the chart. Examples here use release `inlets-uplink`, namespace `inlets`, tenant namespace `tunnels`, and illustrative domains; adapt them consistently.

## Establish the deployment choices

Inspect the kube context, existing Helm releases, ingress classes/controllers, cert-manager, namespaces, and LoadBalancer addresses. Reuse working infrastructure and respect any GitOps ownership. Do not install another ingress controller or replace an issuer merely because an example uses a different one.

Resolve these choices from the request and cluster; ask only for missing information that affects deployment:

- Target cluster/context, release namespace, and a local file containing an Uplink license. A standalone Inlets Pro license does not work. The license key must be uppercase; normalize a protected copy if needed without printing its contents.
- The client connection hostname, its DNS provider, and how the ingress is reachable from the intended client network. Public HTTP01 requires Internet reachability on port 80; HTTPS/WebSockets require a reachable TLS listener, normally port 443. Existing bare-metal/K3s clusters need a working exposure mechanism; a private LoadBalancer IP or a pending LoadBalancer does not establish public reachability.
- Existing ingress choice, issuer, and ACME contact email. Recommend Traefik for a new ingress setup; use Istio when selected or already adopted by the team.
- Whether application traffic stays private, selected HTTP services become public, or many HTTP services need wildcard routing. If unspecified, explain the private default before adding any public application routes.
- Tenant namespace layout, optional namespace restriction via `tunnelsNamespace`, and an upstream service reachable from the machine where the client will run.

The client-router must be reachable by tunnel clients even when the tunneled applications are private. Public client connections do not imply public application access.

When public ingress is unavailable, especially on a local Slicer VM or a cluster behind NAT, explicitly present both a **self-signed issuer for quick local testing/evaluation only** and a **public IP via `use-inlets-operator`**. Do this before attempting public HTTP01 issuance, unless the user has already selected a path. Also consider DNS01 with a user-controlled domain when clients already have a private network route. Read [references/private-cluster-evaluation.md](references/private-cluster-evaluation.md) for the choices, certificate trust setup, and operator skill link. A self-signed certificate or DNS01 solves certificate issuance, not network reachability; the local evaluation option is not a production setup.

| Choice | Application access | Additional setup |
|---|---|---|
| Private (default) | Tunnel ClusterIP Services within the Uplink cluster | No data-router or public application routes |
| Public, per tunnel | Selected HTTP hostnames | Per-host DNS, Ingress or Istio route, and certificate |
| Public, wildcard | Hostnames registered on selected tunnels | data-router, wildcard DNS, DNS01 issuer and certificate |

Read [references/ingress.md](references/ingress.md) for the chosen ingress and TLS path. Read [references/public-tunnels.md](references/public-tunnels.md) only when configuring public application access. Public HTTP routing does not expose arbitrary TCP ports; establish the required TCP listener and protocol separately if requested.

## Prepare dependencies and credentials

Check `kubectl`, Helm with OCI support, and cluster access. Install cert-manager only when absent, using a compatible version of its official chart with CRDs enabled:

```bash
helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager --create-namespace --set crds.enabled=true
```

Wait for its deployments/webhook before creating issuers. Create the Uplink namespace if missing and create the license Secret from the user's protected file:

```bash
kubectl create namespace inlets
kubectl create secret generic inlets-uplink-license -n inlets \
  --from-file=license=./LICENSE_UPLINK
```

Check for existing namespaces and Secrets before running create commands. Preserve existing license and API credentials unless replacement is part of the task. Keep credentials out of chart values, commits, and chat output.

## Inspect, render, and install

Inspect the chart before choosing a version and pin that version for both rendering and installation:

```bash
helm show chart oci://ghcr.io/openfaasltd/inlets-uplink-provider
helm show values oci://ghcr.io/openfaasltd/inlets-uplink-provider
```

Create `values.yaml` using the chosen ingress reference. For private tunnels explicitly keep `dataRouter.enabled: false`; do not enable automatic tunnel ingress. For public access merge the applicable settings into the same values file. Preserve all existing overrides on upgrades.

Set `UPLINK_CHART_VERSION` to the selected published version, then render and review:

```bash
helm template inlets-uplink oci://ghcr.io/openfaasltd/inlets-uplink-provider \
  --version "$UPLINK_CHART_VERSION" --namespace inlets \
  --values ./values.yaml --include-crds
```

Rendered output can contain a generated API token Secret; capture it privately and redact Secrets when reviewing. Verify ingress classes, issuer kinds/names/namespaces, TLS hosts, service ports, API routing, and that only requested application routes are public. Offline rendering cannot preserve a live token through Helm's `lookup`; do not apply the rendered bundle as a substitute for Helm installation.

```bash
helm upgrade --install inlets-uplink \
  oci://ghcr.io/openfaasltd/inlets-uplink-provider \
  --version "$UPLINK_CHART_VERSION" --namespace inlets \
  --values ./values.yaml --wait --timeout 5m
```

The chart is public. If registry authorization fails, diagnose the local Helm registry credentials; do not request a private registry subscription or erase unrelated Docker credentials.

## Verify the control plane and management API

```bash
kubectl get deployments,pods,services -n inlets
kubectl rollout status deployment/uplink-operator -n inlets --timeout=180s
kubectl rollout status deployment/client-router -n inlets --timeout=180s
kubectl get certificates,issuers -n inlets
```

For Kubernetes Ingress inspect `ingress/client-router` and wait for `certificate/client-router-cert` to be Ready. For Istio inspect its Gateway/VirtualService and the certificate in the gateway workload namespace (normally `istio-system`). Check DNS and trusted TLS from the client network; for self-signed evaluations use the explicit certificate trust described in the evaluation reference. Do not use disabled certificate verification as a success criterion.

The chart enables `clientApi` by default. Kubernetes Ingress normally routes `/v1` on the client-router hostname to `client-api:8080`. Verify this route explicitly for Istio; see the ingress reference. Helm generates the `client-api-token` Secret, key `client-api-token`, unless an existing/configured token is used. Retrieve it only into a protected local file when needed. This management token is separate from tunnel connection tokens and application authentication.

For OIDC or a dedicated API hostname, follow the [API setup sections](https://docs.inlets.dev/uplink/installation/#setup-the-rest-api) and [REST API reference](https://docs.inlets.dev/uplink/rest-api/). Check the selected chart's `clientApi` values, DNS, TLS, and authentication. Verify an authenticated read operation; responses can include tunnel tokens, so redact them before sharing. A running Pod alone does not prove API access.

## Connect and test the first tunnel

Follow [Create a tunnel](https://docs.inlets.dev/uplink/create-tunnels/) and [Connect the tunnel client](https://docs.inlets.dev/uplink/connect-tunnel-client/). Keep tenant tunnels out of the management namespace. For a new tenant namespace:

```bash
kubectl create namespace tunnels
kubectl label namespace tunnels inlets.dev/uplink=1
kubectl create secret generic inlets-uplink-license -n tunnels \
  --from-file=license=./LICENSE_UPLINK
```

For the documented Istio sidecar setup also label the tenant namespace `istio-injection=enabled` before creating workloads. Respect an existing mesh revision/injection policy. Create a `Tunnel` with an automatically generated connection token:

```yaml
apiVersion: uplink.inlets.dev/v1alpha1
kind: Tunnel
metadata:
  name: sample
  namespace: tunnels
spec:
  licenseRef:
    name: inlets-uplink-license
    namespace: tunnels
```

Inspect `inlets-pro tunnel --help` and `inlets-pro tunnel connect --help`; install the plugin with `inlets-pro plugin get tunnel` if missing. Generate the connection instructions:

```bash
inlets-pro tunnel connect sample --namespace tunnels \
  --domain uplink.example.com --upstream http://127.0.0.1:8080
```

The command contains a secret: capture it privately. Run the generated `inlets-pro uplink client` command on the host that can reach the upstream. Its URL identifies the control-plane domain and `/namespace/tunnel`. Never substitute a public application hostname for `--domain`. For persistence, inspect the plugin's `--format systemd` or `--format k8s_yaml` output before installing it on the intended client host/cluster.

Verify the upstream locally, a successful client connection, and a real request from inside the Uplink cluster to `http://sample.tunnels:8000`. For host-mapped upstreams supply the corresponding Host header. For TCP, declare the selected ports in the Tunnel and test the actual protocol against its Service. See [private access](https://docs.inlets.dev/uplink/private-tunnels/).

If public access was selected, complete the public-tunnels reference and test the public hostname too. Report the deployed release/version, context, domains, visibility, values location, credential file locations (no values), and successful checks. Distinguish a ready control plane from an unverified tunnel if client access or DNS is still unavailable.

## Troubleshooting and upgrades

Use [Uplink troubleshooting](https://docs.inlets.dev/uplink/troubleshooting/) for observed failures. Check events and logs for `uplink-operator`, `client-router`, the tenant tunnel deployment, and `data-router` when enabled. Missing tunnel resources commonly require checking the namespace label, operator scope, local license Secret/reference, and license capacity. TLS failures require checking DNS, ACME challenges, solver class, and issuer namespace. Connection failures require checking the control-plane URL, token, WebSocket handling/timeouts, and client/server versions.

For upgrades, preserve release values and token ownership, render the chosen version, and compare CRDs. Helm does not upgrade existing CRDs automatically; extract and review the CRD from that same chart version before applying a needed CRD update. Changes to `inletsVersion` can restart existing tunnel servers through the operator; account for that interruption. Keep an explicit ingress class on legacy installations so a chart default change does not switch their controller.
