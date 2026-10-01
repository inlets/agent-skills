# Verify the management API

Read this when validating an installation. Keep tests bounded and use the context and hostname already chosen by the user. Tunnel/client verification belongs to `use-inlets-uplink`.

## Management API

Use the documented [list tunnels endpoint](https://docs.inlets.dev/uplink/rest-api/#list-tunnels), `GET /v1/tunnels`, with an optional `namespace` query. Check HTTP 200 with valid authentication and HTTP 401 without a token for the default static-token configuration. Do not discover endpoints by probing `/v1/`, `/v1/health`, or guessed readiness paths.

Keep the token and response body in protected files; a tunnel listing can contain connection credentials. Build a protected Authorization header file from the retrieved token, then pass its path using curl's `--header @file` support. Do not expand the token into curl's process arguments or print the response body. Report the status and non-secret evidence needed for the check.

For example, after setting `UPLINK_API_URL` to the actual HTTPS base URL and preparing `api-authorization.header` in a private directory:

```bash
umask 077
curl --silent --show-error --connect-timeout 5 --max-time 20 \
  --header @./api-authorization.header \
  --output ./api-tunnels.json --write-out '%{http_code}\n' \
  "$UPLINK_API_URL/v1/tunnels"
```

Add `--cacert ./uplink-eval-cert.pem` for a self-signed evaluation. `--resolve hostname:443:reachable-ip` can test the correct TLS hostname before local DNS is configured; it does not change DNS for the tunnel client. Run the same request without the header file to test rejection.
