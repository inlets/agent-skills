# Client placement and end-to-end verification

Use this for a first tunnel test, including a handoff from `setup-uplink`. Use the chosen context, tenant namespace, client host, and upstream; creating a tunnel is not part of control-plane readiness alone.

## Prepare the client

Prefer a host already in scope that has `inlets-pro` and can reach both ingress and the upstream. A client on a separate host exercises more of the network path than one colocated with the Uplink server. Do not inspect or modify unrelated VMs just because they appear in a VM listing. If no upstream was provided for an evaluation, a temporary loopback HTTP server serving a distinct marker is sufficient; bind it to `127.0.0.1`.

Generate connection instructions in an environment with appropriate cluster access. The runtime client needs its tunnel URL, token, upstream mappings, and any custom TLS trust file, not administrator credentials or the management plugin. When installation is necessary, choose a version compatible with the tunnel server, preferably the deployed `inletsVersion`, and verify it afterward.

Follow [the connection workflow](../SKILL.md#connect-and-verify) to derive the URL privately and retrieve the token with `inlets-pro tunnel token` into a protected file. Build an explicit client invocation using those values and `--token-file`, keeping credentials out of command arguments and output.

Check hostname resolution, local upstream response, and TLS trust before starting the client. When using a self-signed/private certificate, include [the optional `--tls-ca` flag](../SKILL.md#optional-trust-for-self-signed-or-private-certificates) on the first attempt. After connection, inspect the actual tunnel Service/endpoints and request the marker through that Service.

## Collect a real in-cluster response

Reuse an appropriate existing diagnostic Pod or create a uniquely named temporary Pod. For an HTTP smoke test, set `UPLINK_TEST_URL` to the observed tunnel Service address and port, and `UPLINK_TEST_CURL_IMAGE` to a suitable curl image/tag. Check that the example Pod name is unused. This pattern attaches to collect the response and propagates curl failures:

```bash
kubectl run uplink-http-check -n "$TUNNEL_NS" --attach=true --rm \
  --restart=Never --pod-running-timeout=90s \
  --image="$UPLINK_TEST_CURL_IMAGE" --command -- \
  curl --fail-with-body --silent --show-error --max-time 15 \
  "$UPLINK_TEST_URL"
```

Add a Host header when selecting a mapped upstream. If attachment is unavailable, create a Pod that stays alive, wait for Ready with a timeout, execute curl, collect output/exit status, then delete that exact temporary Pod. Do not delete a just-created one-shot Pod before collecting its result. A created Pod or a logged connection is not a successful upstream response.

## Report coverage and clean up

Record which machine ran the client, which ran the upstream, which issued the verification request, and whether any port-forward was involved. A colocated client verifies the local tunnel path; it does not prove connectivity from a separate customer network. API access from the workstation does not fill that gap.

Track temporary client/upstream processes and port-forwards by PID or the relevant process manager's identifier. Stop test-only processes and remove only the temporary diagnostic resources when complete, unless the user asked to keep the demo running. Preserve the installed Uplink release and intended tunnels. If a demo remains running, report its exact location, how to inspect/stop it, and any hostname entries or credential files it depends on.
