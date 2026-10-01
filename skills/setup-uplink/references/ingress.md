# Control-plane ingress and TLS

Source: [Uplink installation](https://docs.inlets.dev/uplink/installation/). The examples below were checked against OCI chart `inlets-uplink-provider` version `0.6.4`; inspect the selected version again during deployment.

If the cluster has no public ingress route, first follow [private cluster evaluation](private-cluster-evaluation.md). The ACME examples below assume public HTTP01 reachability; the evaluation reference supplies a self-signed alternative and a public-IP option using the inlets operator.

## Kubernetes Ingress: Traefik or an existing controller

Reuse an existing controller when suitable. For a new Traefik installation:

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update traefik
helm install traefik traefik/traefik --namespace traefik --create-namespace
```

For public ingress, wait for a publicly reachable LoadBalancer address and configure the client-router hostname's A/AAAA or CNAME record accordingly. Publish AAAA only when IPv6 actually reaches the ingress. Ensure external TCP 80 reaches the HTTP01 solver and 443 reaches the TLS/WebSocket listener. For a private evaluation, use the selected private route or local port-forward and its matching hostname instead.

Use this as the Kubernetes Ingress base for `values.yaml`:

```yaml
ingress:
  class: traefik
  issuer:
    enabled: true
    name: letsencrypt-prod
    email: admin@example.com
clientRouter:
  domain: uplink.example.com
  tls:
    issuerName: letsencrypt-prod
    ingress:
      enabled: true
    istio:
      enabled: false
dataRouter:
  enabled: false
```

For another controller set `ingress.class` to its actual IngressClass. Check the rendered issuer's HTTP01 solver and the controller's WebSocket and idle timeout configuration. An arbitrary class change alone does not establish compatibility.

For an existing or staging issuer set `ingress.issuer.enabled: false` and `clientRouter.tls.issuerName` to its name. Verify the rendered reference kind: chart 0.6.4 uses a namespaced `Issuer`, so that issuer belongs in `inlets`. A `ClusterIssuer` with the same name is not interchangeable; use a supported chart option or explicitly managed ingress/certificate resources if needed. Staging certificates are not publicly trusted.

### Existing ingress-nginx installations

The [installation docs](https://docs.inlets.dev/uplink/installation/#ingress-nginx) identify community ingress-nginx as retired and the chart's default as Traefik starting in 0.5.0. Prefer supported controllers for new installations. Preserve a user's explicitly selected legacy controller during an Uplink upgrade; migration is a separate change.

For retained ingress-nginx set `ingress.class: nginx` and add these under `clientRouter.tls.ingress.annotations`:

```yaml
nginx.ingress.kubernetes.io/limit-connections: "300"
nginx.ingress.kubernetes.io/limit-rpm: "1000"
nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
nginx.ingress.kubernetes.io/keepalive-timeout: "350"
nginx.ingress.kubernetes.io/proxy-buffer-size: "128k"
```

## Istio

Use the existing mesh's installation and injection convention. If installing Istio is part of the requested setup, the Uplink docs offer `arkade install istio`; check compatibility with the cluster first. For the documented sidecar setup label `inlets` and each tenant namespace before creating Pods:

```bash
kubectl label namespace inlets istio-injection=enabled --overwrite
```

The chart creates an Istio `networking.istio.io` Gateway and VirtualService, not Kubernetes Gateway API resources. Check the served Istio API versions, gateway workload selector, namespace, LoadBalancer, and HTTP01 handling.

Use this Istio base:

```yaml
ingress:
  class: istio
  istio:
    enabled: true
  issuer:
    enabled: true
    name: letsencrypt-prod
    email: admin@example.com
clientRouter:
  domain: uplink.example.com
  tls:
    issuerName: letsencrypt-prod
    ingress:
      enabled: true
    istio:
      enabled: true
dataRouter:
  enabled: false
```

In chart 0.6.4, `ingress.istio.enabled` places the generated Issuer in `istio-system` and selects the Istio ACME solver. Setting only `ingress.issuer.class: istio` from the documentation example has no effect in this chart. The Certificate is also generated in `istio-system`, and the Gateway selects `istio: ingressgateway` with `credentialName: client-router-cert`. Check that these assumptions match the existing mesh; use explicitly managed resources when they do not.

In this chart, keep `clientRouter.tls.ingress.enabled: true` alongside `clientRouter.tls.istio.enabled: true`: the Istio flag suppresses the Kubernetes Ingress, while the ingress flag includes the hostname in the generated issuer's `selector.dnsNames`. Verify both results when rendering. Alternatively provide an existing Issuer with an explicit solver for the hostname and disable chart issuer generation.

### Route the API with Istio

Chart 0.6.4's generated VirtualService routes all paths to `client-router:8080`; it omits the `/v1` route that the Kubernetes Ingress template supplies. Before claiming that the API is available, use a maintained Helm post-renderer or the cluster's declarative overlay mechanism to add the API route before the catch-all:

```yaml
http:
  - match:
      - uri:
          prefix: /v1
    route:
      - destination:
          host: client-api.inlets.svc.cluster.local
          port:
            number: 8080
  - match:
      - uri:
          prefix: /
    route:
      - destination:
          host: client-router.inlets.svc.cluster.local
          port:
            number: 8080
```

Preserve the path prefix. Retain this customization on upgrades. Alternatively manage a dedicated API hostname with its own Istio TLS/route resources. `clientApi.tls.ingress.enabled` creates a Kubernetes Ingress; it does not create an Istio VirtualService. Keep management authentication enabled in either case.
