# Uplink APIs and billing

## Client API

The Uplink client API manages tunnels and labeled tenant namespaces. It normally shares the client-router domain beneath `/v1`, unless installation values configure a separate API domain.

Authentication supports either the static client API token or an OAuth access token. Retrieve the static administrator token without displaying it:

```bash
umask 077
CLIENT_API_TOKEN_FILE="$(mktemp)"
CLIENT_API_HEADER_FILE="$(mktemp)"
trap 'rm -f "$CLIENT_API_TOKEN_FILE" "$CLIENT_API_HEADER_FILE"' EXIT
kubectl get secret client-api-token -n inlets \
  -o jsonpath='{.data.client-api-token}' \
  | base64 --decode > "$CLIENT_API_TOKEN_FILE"
printf 'Authorization: Bearer ' > "$CLIENT_API_HEADER_FILE"
cat "$CLIENT_API_TOKEN_FILE" >> "$CLIENT_API_HEADER_FILE"
printf '\n' >> "$CLIENT_API_HEADER_FILE"
export CLIENT_API="https://uplink.example.com"
```

Use Secret metadata only during discovery. Do not decode or preview any credential unless it is required for an authorized request, and never print even a prefix. Send `Authorization: Bearer ...`. Prefer `--data @payload.json` for non-trivial JSON, and never enable shell tracing around secrets.

| Operation | Method and path |
|---|---|
| Get tunnel | `GET /v1/tunnels/{name}?namespace={namespace}` |
| Get tunnel with metrics | `GET /v1/tunnels/{name}?namespace={namespace}&metrics=1` |
| List tunnels | `GET /v1/tunnels?namespace={namespace}` |
| Create tunnel | `POST /v1/tunnels` |
| Update tunnel | `PUT /v1/tunnels` |
| Delete tunnel | `DELETE /v1/tunnels/{name}?namespace={namespace}` |
| List Uplink namespaces | `GET /v1/namespace` |
| Create namespace | `POST /v1/namespace` |
| Delete namespace | `DELETE /v1/namespace/{name}` |

Example tunnel payload:

```json
{"name":"customer-edge","namespace":"tenant-name","tcpPorts":[80,443]}
```

Example request:

```bash
export TUNNEL_NS="tenant-name"
export TUNNEL_NAME="customer-edge"
curl --fail-with-body \
  --header @"$CLIENT_API_HEADER_FILE" \
  "$CLIENT_API/v1/tunnels/$TUNNEL_NAME?namespace=$TUNNEL_NS&metrics=1"
```

For OAuth client credentials, obtain the token from the configured identity provider; do not assume its token URL, scopes, or client authentication method. Confirm those with the Uplink administrator.

## Billing Management API

The Billing Management API is for approved customers automating subscription capacity. It does not create tunnels. Access requires a separate agreement and a billing API key issued by the inlets team; one key is tied to one subscription.

Obtain the billing API base URL from the customer's agreement or Uplink administrator. Do not derive it from the client-router domain and do not probe guessed routes. In particular, never use a product license against an inferred `/api/...` billing endpoint.

Keep these credentials distinct:

| Credential | Purpose |
|---|---|
| Product license key | Runs inlets Pro/Uplink and queries license entitlement |
| Uplink client API token/OAuth token | Manages tunnels and namespaces |
| Tunnel token | Authenticates one tunnel's client to its server |
| Billing API key | Reads or updates one subscription's billable quantity |

To request a billing key, run `inlets-pro seal keygen`, send only the printed public key to the inlets team, and keep the private key local. Unseal the returned value with `inlets-pro unseal`, then store it in a KMS, Vault, or password manager. Rotate it if exposed.

| Operation | Authentication |
|---|---|
| `GET /v1/billing/quantity` | Billing API key |
| `PUT /v1/billing/quantity` with `{"quantity": N}` | Billing API key |
| `GET /v1/license/inlets/entitlement` | Product license key |

Explain quantities before updating them: Uplink provider capacity may round requested tunnels to subscription bands, while standalone inlets Pro may bill per tunnel after a minimum. Read the GET response's `included_quantity` and `quantity_step`; show the requested and resulting billable quantities. Do not perform a billing update without explicit user authorization.

Official page: https://docs.inlets.dev/uplink/billing-management-api/
