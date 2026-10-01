# Private clusters and local evaluations

Use this when the Uplink cluster lacks public ingress. Inspect reachability from the intended client host, not only from inside the cluster. A populated Service `EXTERNAL-IP` may still be private, and a reachable Kubernetes API does not prove that ingress ports are reachable.

Present the applicable choices before installing a public ACME issuer. Reuse a choice already made by the user.

| Option | Suitable use | Requirements |
|---|---|---|
| Self-signed issuer | Local testing and evaluation; not production guidance | Private route or local port-forward, test hostname, explicit client certificate trust |
| Public IP with `use-inlets-operator` | Test real Internet ingress or provide public client connectivity | Cloud VM/provider credentials, Inlets Pro license, DNS, and public ports 80/443 |
| DNS01 with a real domain | Trusted certificates while ingress stays private | Domain/DNS control, DNS01 issuer, and an existing private route for clients |

Always include the first two options in this scenario, identifying the self-signed option as evaluation-only. DNS01 is useful when the user controls a domain; it does not make a private endpoint publicly reachable. Neither does changing issuer type. Local hostname resolution alone cannot satisfy public HTTP01 validation.

## Self-signed evaluation

If no router hostname was specified, **ask the user to confirm a suggested hostname or provide their own** before writing the values or issuing the certificate. Suggest [nip.io](https://nip.io/) using the ingress IP reachable from the intended client, for example `uplink.192.168.1.50.nip.io` for `192.168.1.50`. It resolves the embedded IP without user-managed DNS records or `/etc/hosts` changes. Use the discovered address, not this example IP, and preserve an already chosen hostname such as `uplink.internal`.

For a loopback-bound port-forward, suggest `uplink.127.0.0.1.nip.io` instead; it works only on the host running that forward. Check resolution through the intended client's resolver: public DNS access is required, and DNS-rebinding protection may block private/loopback answers. If unavailable or unwanted, offer a user-managed hostname or `uplink.test` with scoped local DNS/hosts configuration; do not disable resolver protections. `nip.io` supplies DNS only, not a network route or trusted TLS. Keep the self-signed issuer and explicit certificate trust for this mode.

Use cert-manager's [SelfSigned issuer](https://cert-manager.io/docs/configuration/selfsigned/) for this local evaluation setup. This configuration is not intended as production guidance. Create an Issuer in `inlets` for Kubernetes Ingress:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: uplink-eval-selfsigned
  namespace: inlets
spec:
  selfSigned: {}
```

Merge this override into the chosen ingress values; replace the example domain with the confirmed hostname and retain the controller-specific configuration:

```yaml
ingress:
  issuer:
    enabled: false
clientRouter:
  domain: uplink.192.168.1.50.nip.io
  tls:
    issuerName: uplink-eval-selfsigned
dataRouter:
  enabled: false
```

For Kubernetes Ingress, create only the Issuer: the chart's annotated Ingress triggers cert-manager to create `client-router-cert` after Helm installation, as described in [ingress.md](ingress.md#who-creates-the-certificate). Do not wait for that Certificate before the Ingress exists. For Istio, create the Issuer in the same namespace as the chart's explicit Certificate, normally `istio-system`, and retain the Istio settings from the ingress reference. For a separate API hostname, configure its issuer reference and hostname too. Render to ensure no production ACME Issuer is created and the Ingress annotation or explicit Certificate references the evaluation Issuer.

Verify the confirmed hostname resolves to an ingress address reachable by each intended client, and use it consistently in values, certificate DNS names, and connection URLs. If the cluster ingress is not directly reachable from the client host, forward the ingress controller's Service TLS port through the kube API, for example local port 8443 to Service port 443. Discover the actual Service name/namespace and use a loopback-bound `kubectl port-forward` on the host running the test client. With the loopback hostname confirmed above, the endpoint is `wss://uplink.127.0.0.1.nip.io:8443/...`. Keep the forward running during testing. Forward the ingress controller, so TLS termination and hostname routing are exercised.

After issuance, export only the public certificate (`tls.crt`) from `client-router-cert` to a local PEM file, using the certificate's actual namespace. Never export `tls.key`. Transfer that certificate through the trusted administrative connection to the test client host.

Use `curl --cacert ./uplink-eval-cert.pem` for HTTPS/API checks. The URL hostname must match the certificate's DNS names. Prefer trust scoped to the test command over modifying the machine-wide trust store. Keep API authentication enabled.

Hand off client setup to [use-inlets-uplink](../../use-inlets-uplink/SKILL.md), supplying the reachable hostname/port and public certificate file. That skill adds the optional `--tls-ca` flag for this self-signed setup before the first connection.

Verify Certificate readiness, trusted HTTPS, and API authentication. Report this as a local evaluation, not public reachability or production readiness; report any separate tunnel test performed through `use-inlets-uplink`. Record any port-forward dependency and local hostname entry. A reissued self-signed certificate may require updating the trusted PEM. For production, move to a suitable managed/public issuer or the organisation's managed PKI and verify the intended network path again.

## Public IP through the inlets operator

Offer the companion [use-inlets-operator skill](https://github.com/inlets/agent-skills/blob/master/skills/use-inlets-operator/SKILL.md). If installed, load it before carrying out that option; otherwise use its linked instructions and the [operator documentation](https://docs.inlets.dev/reference/inlets-operator/). It is optional for Uplink and is distinct from Uplink's own `uplink-operator` component.

Use it to expose the existing ingress controller's LoadBalancer Service on ports 80/443 via a cloud exit VM. Keep TLS termination at the cluster ingress. Explain the cloud VM cost and separate Inlets Pro credential/license requirements before provisioning; an Uplink license is not a substitute. Selecting a local evaluation does not authorize provisioning public infrastructure.

Scope the operator to the intended ingress Service. Where K3s ServiceLB, MetalLB, or another LoadBalancer controller exists, follow the companion skill's `annotatedOnly: true` and Service annotation guidance; preserve other Services. Qualify the two distinct CRDs as `tunnels.operator.inlets.dev` and `tunnels.uplink.inlets.dev` when inspecting them.

After the public IP is provisioned, point the client-router DNS name at it, verify Internet reachability on 80/443, and return to the normal Uplink ingress/TLS setup. Public control-plane access still leaves application tunnels private unless public data-plane routes were selected separately. Before disposing of an evaluation cluster, follow the operator skill's cleanup procedure so its cloud VM is not orphaned.
