# Public application access

Source: [Expose tunnels publicly](https://docs.inlets.dev/uplink/expose-tunnels/). Apply this only for selected services the user wants public. Preserve private tunnels and establish a working private connection before diagnosing public routing.

This reference configures the ingress infrastructure. Use [use-inlets-uplink](../../use-inlets-uplink/SKILL.md) to create/connect the target tunnels and return their actual Service names, namespaces, ports, and host mappings before applying per-tunnel routes.

These paths expose HTTP applications with HTTPS at the ingress. They do not provide arbitrary TCP forwarding or TLS passthrough to a remote ingress. Tunnel connection tokens and management API tokens do not authenticate visitors to public applications; configure the intended application/ingress authentication separately.

## Per-tunnel Kubernetes Ingress

Use individual hostnames for a modest number of services or independent/custom domains. Point each hostname at the controller's address. Create an HTTP01 Issuer in the **tunnel namespace**, with a solver for the chosen ingress class, or reference an existing ClusterIssuer using the corresponding annotation. A namespaced Issuer in `inlets` cannot issue a certificate for an Ingress in `tunnels`.

The control-plane chart's generated issuer can restrict its solver to control-plane hostnames. Use a suitable application issuer instead of assuming that issuer covers every public domain. Ensure `spec.acme.privateKeySecretRef.name` is nested correctly when adapting the upstream example.

For a connected `sample` tunnel in `tunnels`, after provisioning a namespaced issuer called `tunnels-letsencrypt-prod`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-public
  namespace: tunnels
  annotations:
    cert-manager.io/issuer: tunnels-letsencrypt-prod
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - app.example.com
      secretName: sample-public-cert
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sample
                port:
                  number: 8000
```

The Ingress and backend Service must share a namespace. Match the host to the client's upstream mapping when forwarding multiple HTTP services. `dataRouter.enabled` remains false for this approach. Chart versions with `operator.tunnelIngress` offer automation; inspect their configuration and CRD semantics before enabling it, and avoid creating duplicate routes manually.

## Per-tunnel Istio Gateway

Use an Istio Gateway with HTTPS termination and a VirtualService for each selected hostname (or a shared Gateway with multiple hosts). Create the Issuer, Certificate and TLS Secret in the gateway workload namespace, normally `istio-system`. Set Gateway `credentialName` to that Secret and use the actual gateway selector.

Route each hostname to the fully qualified tenant Service, for example `sample.tunnels.svc.cluster.local:8000`. Match `VirtualService.hosts` to the Gateway host and reference the Gateway by `namespace/name` if it is in another namespace. Enable the mesh's injection convention in tenant namespaces. Verify certificates and routes independently of the client-router's Gateway.

## Wildcard data-router

Use this for many HTTP hostnames within a common domain. Establish the user's wildcard suffix and DNS provider; do not assume a provider or request raw API credentials in chat.

1. Create wildcard DNS such as `*.apps.example.com` pointing to the public ingress. This record does not replace the separate client connection record `uplink.example.com`.
2. Configure a cert-manager DNS01 Issuer using the provider's [supported solver](https://cert-manager.io/docs/configuration/acme/dns01/). Store appropriately scoped DNS credentials in a Secret where the issuer expects them. Wildcard certificates require DNS01, not HTTP01.
3. For Kubernetes Ingress, put the DNS01 Issuer in `inlets`, merge these values into the existing Uplink values, then render and upgrade the same release/version:

```yaml
dataRouter:
  enabled: true
  wildcardDomain: apps.example.com
  tls:
    issuerName: inlets-wildcard
    ingress:
      enabled: true
```

Do not include `*.` in `wildcardDomain`. The chart creates a wildcard Ingress and requests `*.apps.example.com`; the named Issuer must already exist. A single-label wildcard does not cover deeper names such as `app.tenant.apps.example.com`.

For **Istio**, enable `dataRouter.enabled` but leave `dataRouter.tls.ingress.enabled: false`. Chart 0.6.4 has no data-router Istio Gateway template or `dataRouter.tls.istio.enabled` option. Create a DNS01 Issuer and wildcard Certificate in the gateway workload namespace, then an Istio Gateway/VirtualService for `*.apps.example.com` routing to `data-router.inlets.svc.cluster.local:8080`. Reference the wildcard TLS Secret with `credentialName`. Keep these declarative resources alongside the Helm values.

4. Hand the wildcard suffix and chosen public hostnames to `use-inlets-uplink` for tunnel domain registration and matching client upstreams. Adding wildcard DNS alone does not register a tunnel: `spec.ingressDomains`, the requested Host, and the client upstream mapping must agree. Keep the client connection hostname distinct from these application domains, and do not register private tunnels for public access.

## Verify the selected exposure

Check public DNS, Certificate readiness, and an HTTPS request returning the expected upstream response. Verify intended visitor authentication, if configured. For wildcard routing inspect `data-router` logs to confirm the requested host resolves to the intended tenant Service. Test an unregistered hostname to check it does not reach an unintended application, and verify private tunnels have no public routes or domain registration.

On failure follow the path from ingress or Gateway to data-router (if used), tenant Service/endpoints, tunnel server, connected client, and local upstream. A valid certificate or an ingress 404 alone does not establish a working tunnel.
