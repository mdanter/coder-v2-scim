## Debugging Okta SCIM "Error authenticating: null" (RKE/Kubernetes Deployment)

The `Error authenticating: null` error means Okta sent a request to Coder's SCIM endpoint and got back an authentication failure (or couldn't connect at all). Work through these steps in order:

### 1. Verify the SCIM endpoint is reachable from the internet

From a machine outside your network:

```bash
curl -v https://<FQDN>/scim/v2/ServiceProviderConfig
```

- **Connection timeout / DNS failure**: Okta can't reach your Coder instance. The URL must be publicly accessible or accessible from Okta's IP ranges. Check your RKE ingress controller and any firewall rules.
- **200 with JSON**: Endpoint is reachable and SCIM is enabled.
- **404**: SCIM is not enabled or the URL path is wrong.

### 2. Verify the Bearer token works

```bash
curl -v -H "Authorization: Bearer YOUR_SCIM_TOKEN" \
  https://<FQDN>/scim/v2/Users
```

- **200**: Auth is working.
- **401**: Token mismatch or `CODER_SCIM_AUTH_HEADER` is not set.

Confirm the token on the Coder pod:

```bash
# Find the coder pods (release name may differ if you used a custom name)
kubectl get pods -n <coder-namespace> -l app.kubernetes.io/name=coder

# Check the env var is set and has the expected value
kubectl exec -it <coder-pod-name> -n <coder-namespace> -- env | grep CODER_SCIM_AUTH_HEADER

# Check your Helm values (replace "coder" with your Helm release name if different)
helm get values <release-name> -n <coder-namespace> | grep -i scim
```

The token is set either directly in `coder.env` in your Helm values, or via a Secret referenced by `coder.envFrom`. If using `envFrom`, check the Secret you configured:

```bash
kubectl get secret <your-secret-name> -n <coder-namespace> -o jsonpath='{.data.CODER_SCIM_AUTH_HEADER}' | base64 -d
```

### 3. Check the Okta Authorization field format (most common cause)

Okta's "HTTP Header" auth mode **automatically prepends `Bearer `** to whatever you enter. The field in Okta is labeled "Bearer:" — it's a prefix, not something you type.

- **If you entered `Bearer <token>`**, Okta sends `Authorization: Bearer Bearer <token>` — this will fail.
- **Enter ONLY the raw token** in the Okta Authorization field, without the word "Bearer".

This is the #1 cause of this exact error.

### 4. Check for trailing whitespace or newlines in the token

Copy-paste can introduce invisible characters. Re-generate a clean token:

```bash
openssl rand -hex 32
```

Update it in your Helm values and upgrade:

```bash
# If the token is set directly in coder.env in values.yaml, update it there, then:
helm upgrade <release-name> coder/coder -n <coder-namespace> -f values.yaml

# If the token is in a Secret referenced by coder.envFrom, update that Secret,
# then restart the deployment to pick up the change:
kubectl rollout restart deployment/coder -n <coder-namespace>
```

Paste the exact same new token value into Okta's Authorization field.

### 5. Check Coder pod logs

```bash
# Tail logs from the deployment
kubectl logs deployment/coder -n <coder-namespace> --follow | grep -i scim
```

Look for:
- `"invalid authorization"` — token mismatch
- Any `401` responses to `/scim/v2` endpoints
- **No SCIM log lines at all** — Okta isn't reaching the server (network/ingress issue)

If multiple replicas are running, check all pods:

```bash
kubectl logs -l app.kubernetes.io/name=coder -n <coder-namespace> --all-containers | grep -i scim
```

### 6. Verify the RKE Ingress is passing the Authorization header

This is a common issue with Kubernetes ingress controllers. Check your Ingress resource:

```bash
kubectl get ingress -n <coder-namespace> -o yaml
```

**For nginx ingress controller** (common on RKE), ensure no annotations are stripping or consuming the `Authorization` header. If you're using `nginx.ingress.kubernetes.io/auth-url` for external auth, it may consume the `Authorization` header before it reaches Coder.

Test from inside the cluster to bypass the ingress entirely:

```bash
# Get the Coder service
kubectl get svc -n <coder-namespace>

# Exec into a temporary pod and curl the service directly
kubectl run -it --rm debug --image=curlimages/curl -n <coder-namespace> -- \
  curl -v -H "Authorization: Bearer YOUR_TOKEN" \
  http://coder.<coder-namespace>.svc.cluster.local/scim/v2/ServiceProviderConfig
```

If this works but the external URL doesn't, the problem is in the ingress or load balancer layer.

### 7. Verify TLS/certificate validity

```bash
curl -vI https://<FQDN>/scim/v2/ServiceProviderConfig
```

Check that:
- Certificate is valid and not expired
- Certificate chain is complete (no missing intermediates)
- Domain matches the cert's SAN

If you're using cert-manager to manage TLS certificates:

```bash
kubectl get certificates -n <coder-namespace>
kubectl describe certificate <cert-name> -n <coder-namespace>
```

If TLS is terminated at the ingress via `coder.tls.secretNames` in your Helm values, verify the referenced Secret exists and is valid:

```bash
kubectl get secret <tls-secret-name> -n <coder-namespace>
```

Okta will silently reject self-signed or invalid certs and return a `null` error.

### 8. Check for NetworkPolicies blocking traffic

RKE deployments sometimes have NetworkPolicies that restrict traffic:

```bash
kubectl get networkpolicies -n <coder-namespace>
```

Ensure nothing is blocking inbound traffic to the Coder pods on port 8080 (the default HTTP port in the Helm chart).

### 9. Check Okta System Logs

In Okta Admin Console: **Reports > System Log**

Filter for your SCIM app. The `null` in the error often means Okta got no response body — pointing to a network-level issue (timeout, TLS error, connection refused).

---

**Summary of most likely causes, in order:**

1. **Double "Bearer" prefix** — enter only the raw token in Okta, not `Bearer <token>`
2. **Ingress stripping the Authorization header** — test by curling the Coder service directly inside the cluster
3. **Token mismatch** — verify the env var value on the running pod matches what's in Okta
4. **TLS certificate issue** — Okta won't connect to endpoints with invalid certs
