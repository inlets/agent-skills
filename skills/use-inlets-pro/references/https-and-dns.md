# Automated HTTPS and DNS

## Preconditions

Automated HTTPS requires:

- a public server reachable on TCP 80 and 443 for the HTTP data plane and HTTP-01 challenge;
- the control port, normally 8123, reachable by the private tunnel client;
- each `--letsencrypt-domain` resolving to the public server;
- no conflicting web server bound to 80/443 unless it is deliberately part of the architecture.

Do not start production certificate issuance until DNS resolves correctly from public resolvers. Use `--letsencrypt-issuer staging` during setup, then switch to `prod`.

## Headless DNS workflow

First discover the hosted zone and current records using the provider's CLI. Show the proposed mutation and ask for confirmation before changing a real domain unless the user already explicitly requested that mutation.

Generic verification:

```bash
dig +short A app.example.com
dig +short AAAA app.example.com
```

Remove a stale AAAA record if the server has no working IPv6 path; ACME validation may follow it.

### DigitalOcean (`doctl`)

```bash
doctl compute domain records list example.com
doctl compute domain records create example.com \
  --record-type A --record-name app --record-data 203.0.113.10
```

### Cloudflare (`cloudflared` is not a DNS record CLI)

Use the Cloudflare API, Terraform, or an installed supported CLI. With the API, resolve the zone ID first and use a narrowly scoped token with DNS edit permission. Do not guess a zone ID or overwrite an existing record blindly.

```bash
curl --fail-with-body \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones?name=example.com"
```

Create/update `type=A`, `name=app.example.com`, and the public IP through the returned zone's DNS-record endpoint. For HTTP-01, disable Cloudflare proxying initially (`proxied: false`) unless the deployed setup is known to support it.

### AWS Route 53 (`aws`)

```bash
aws route53 list-hosted-zones-by-name --dns-name example.com
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE_ID \
  --change-batch file://change.json
```

Use an UPSERT change containing an `A` record, a deliberate TTL, and the server IP. Wait for `aws route53 get-change --id CHANGE_ID` to report `INSYNC`.

### Google Cloud DNS (`gcloud`)

```bash
gcloud dns managed-zones list
gcloud dns record-sets create app.example.com. \
  --zone=ZONE --type=A --ttl=300 --rrdatas=203.0.113.10
```

### Azure DNS (`az`)

```bash
az network dns zone list -o table
az network dns record-set a add-record \
  --resource-group DNS_RESOURCE_GROUP \
  --zone-name example.com --record-set-name app \
  --ipv4-address 203.0.113.10
```

If no suitable authenticated CLI is available, tell the user exactly what to provide to their DNS administrator: zone, fully qualified hostname(s), record type `A` (and `AAAA` only with working IPv6), public server IP, desired TTL, and that ports 80/443 must reach that server for Let's Encrypt HTTP-01.

## Verification

```bash
dig +short app.example.com @1.1.1.1
curl -I http://app.example.com
curl -I https://app.example.com
```

Check server logs for ACME errors. Distinguish DNS propagation, blocked port 80, rate limiting, and a hostname mismatch rather than repeatedly restarting issuance.
