# Envoy as an OAuth2 proxy (Entra ID) on AKS

**Azure App Service** has a simple out-of-the-box option to add Entra ID
authentication in front of your application before any traffic reaches it (the
"Easy Auth" / Authentication blade), and the same is true for **Azure Container
Apps**. However, **no such built-in option exists for Application Gateway or for
Kubernetes** — there you have to add the authentication layer yourself.

If you need to put Entra ID authentication in front of an app running behind
Application Gateway or on Kubernetes, you can use **Envoy** and its built-in
OAuth2 filter, as shown in this example. See the Envoy OAuth2 filter
documentation:
<https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/oauth2_filter>

---

## Overview

Deployment of the `echo-server` application behind an Envoy proxy that enforces
authentication via **Microsoft Entra ID** (OAuth2 Authorization Code Flow). TLS is
terminated at **ingress-nginx** with a **Let's Encrypt** certificate (cert-manager,
HTTP-01). The external address is built dynamically using **nip.io** (`<IP>.nip.io`).

```
Internet ──HTTPS──▶ ingress-nginx (TLS, Let's Encrypt)
                        │ HTTP
                        ▼
                    Envoy (OAuth2 / Entra ID)
                        │ HTTP
                        ▼
                    echo-server (8081 → 8080)
```

## Prerequisites

- `az` (Azure CLI) — logged in: `az login`
- `kubectl`
- `openssl`
- Permissions to create an App Registration in the Entra ID tenant

> **Important — shell variables:** several steps render config from environment
> variables (`$TENANT_ID`, `$APP_ID`, `$NIP_HOST`, ...). If you open a new terminal
> midway through, those variables are lost and configs get rendered with empty values
> (e.g. an empty `client_id` crashes Envoy with `ClientId: value length must be at
> least 1 characters`). If that happens, re-run step 0 and re-export `APP_ID`/`NIP_HOST`
> from the recovery snippet in step 5 / step 4.

---

## 0. Initial variables

```bash
export TENANT_ID="50ea0418-683d-4007-87fb-c13c8f6b5d0b"
export ACME_EMAIL="your-mail@example.com"          # <-- FILL IN
export RG="envoy-proxy-EntraID-test"               # Resource Group
export AKS="envoy-proxy-EntraID-test"              # AKS cluster name
export LOCATION="westeurope"                        # Azure region
export APP_NAME="envoy-echo-oauth"                  # Entra App Registration name
```

---

## 1. Create the AKS cluster and connect kubectl

```bash
# Resource Group
az group create --name "$RG" --location "$LOCATION"

# AKS cluster (1 node, default VM size, no SSH key)
az aks create \
  --resource-group "$RG" \
  --name "$AKS" \
  --node-count 1 \
  --no-ssh-key

# Fetch kubeconfig and switch context to the new cluster
az aks get-credentials --resource-group "$RG" --name "$AKS" --overwrite-existing

# Verify the connection
kubectl get nodes
```

---

## 2. Namespace + echo-server

```bash
kubectl create namespace echo

# Deployment + Service (port 8081 -> 8080)
kubectl apply -n echo -f https://raw.githubusercontent.com/MariuszFerdyn/k8s-ingress-modsecurity-waf-playground/master/echoserver.yaml
```

---

## 3. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml

# Wait until all 3 deployments are available
kubectl wait --for=condition=Available --timeout=180s \
  -n cert-manager deploy/cert-manager deploy/cert-manager-webhook deploy/cert-manager-cainjector
```

---

## 4. Install ingress-nginx (terminates TLS, provides External IP)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0/deploy/static/provider/cloud/deploy.yaml

kubectl wait --for=condition=Available --timeout=180s \
  -n ingress-nginx deploy/ingress-nginx-controller

# Wait for the External IP (Ctrl-C once it appears in the EXTERNAL-IP column)
kubectl get svc ingress-nginx-controller -n ingress-nginx -w
```

```bash
# Grab the nginx IP and build the nip.io host
NGINX_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export NIP_HOST="${NGINX_IP}.nip.io"
echo "Host = $NIP_HOST"
```

---

## 5. Register the application in Entra ID (idempotent)

This step reuses an existing App Registration if one with the same name already
exists, and **always sets a fresh client secret**. Safe to re-run.

> **Prerequisite:** you must already be logged in to Azure CLI (`az login`) with an
> account that has permission to **create / update App Registrations** in the tenant
> (e.g. Application Administrator, Cloud Application Administrator, or an equivalent
> role / ownership). This step does not call `az login` for you.

```bash
# Verify the CLI session is actually valid before touching anything.
az account show >/dev/null 2>&1 || { echo "Not logged in - run: az login --tenant $TENANT_ID"; }

# Reuse the app if it already exists; otherwise create it.
APP_ID=$(az ad app list --display-name "$APP_NAME" --query "[0].appId" -o tsv)

if [ -z "$APP_ID" ]; then
  echo "App not found - creating a new App Registration..."
  APP_ID=$(az ad app create \
    --display-name "$APP_NAME" \
    --web-redirect-uris "https://$NIP_HOST/callback" \
    --query appId -o tsv)
else
  echo "Reusing existing App Registration: $APP_ID"
  # Make sure the redirect URI matches the current nip.io host.
  az ad app update --id "$APP_ID" \
    --web-redirect-uris "https://$NIP_HOST/callback"
fi

# Stop here if APP_ID is still empty (auth failure, CAE token revoked, etc.)
# BEFORE resetting the secret - otherwise we'd reset a secret on an empty id.
: "${APP_ID:?APP_ID is empty - check 'az login' (CAE/token revoked?) and retry}"
echo "APP_ID=$APP_ID"

# Only now generate a fresh client secret.
CLIENT_SECRET=$(az ad app credential reset \
  --id "$APP_ID" --append --query password -o tsv)
: "${CLIENT_SECRET:?CLIENT_SECRET is empty - secret reset failed}"
echo "CLIENT_SECRET set"
```

> **Recovery (new terminal):** if you lost your shell and need `APP_ID` back without
> creating a new secret:
> ```bash
> APP_ID=$(az ad app list --display-name "$APP_NAME" --query "[0].appId" -o tsv)
> ```
> Note: to rebuild the `envoy-oauth` secret you still need a *secret value*, which is
> only shown at creation time — so in practice, re-running the secret reset above is
> the simplest path.

---

## 6. OAuth2 secret for Envoy

> **Note (HMAC):** `hmac-secret` must be **raw bytes** (hex), not a base64 string.
> The `oauth2` filter decodes the value, and a base64 string ending with `=` causes
> cookie-signing errors. That's why we use `openssl rand -hex 32`.

```bash
HMAC=$(openssl rand -hex 32)

# Recreate cleanly so re-runs pick up the new secret.
kubectl delete secret envoy-oauth -n echo --ignore-not-found
kubectl create secret generic envoy-oauth -n echo \
  --from-literal=client-secret="$CLIENT_SECRET" \
  --from-literal=hmac-secret="$HMAC"
```

---

## 7. ConfigMap with the Envoy configuration

Key parts of the config that turned out to be required:

- **`node.id` / `node.cluster`** — without these, SDS (loading secrets from files) won't start.
- **`auth_type: URL_ENCODED_BODY`** — Entra v2.0 expects the `client_secret` in the token request body.
- **`dns_lookup_family: V4_ONLY`** — `login.microsoftonline.com` also resolves to IPv6;
  the AKS cluster has no IPv6 routing, so without this the code→token exchange fails
  with `Network is unreachable`.

```bash
# Abort early if any required variable is empty (prevents empty client_id -> crash).
: "${TENANT_ID:?set TENANT_ID (step 0)}"
: "${APP_ID:?set APP_ID (step 5)}"

cat > /tmp/envoy.yaml <<EOF
node:
  id: envoy-echo
  cluster: envoy-echo
static_resources:
  listeners:
  - name: main
    address:
      socket_address: { address: 0.0.0.0, port_value: 10000 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: ingress_http
          codec_type: AUTO
          use_remote_address: true
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route: { cluster: echo_backend }
          http_filters:
          - name: envoy.filters.http.oauth2
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.oauth2.v3.OAuth2
              config:
                token_endpoint:
                  cluster: entra_token
                  uri: https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/token
                  timeout: 5s
                authorization_endpoint: https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/authorize
                redirect_uri: "https://%REQ(:authority)%/callback"
                redirect_path_matcher:
                  path: { exact: "/callback" }
                signout_path:
                  path: { exact: "/signout" }
                auth_type: URL_ENCODED_BODY
                credentials:
                  client_id: "${APP_ID}"
                  token_secret:
                    name: token
                    sds_config: { path_config_source: { path: "/etc/envoy/sds/token.yaml" } }
                  hmac_secret:
                    name: hmac
                    sds_config: { path_config_source: { path: "/etc/envoy/sds/hmac.yaml" } }
                auth_scopes: ["openid", "profile", "email", "offline_access"]
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
  clusters:
  - name: echo_backend
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    load_assignment:
      cluster_name: echo_backend
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: echo-server.echo.svc.cluster.local, port_value: 8081 }
  - name: entra_token
    type: STRICT_DNS
    dns_lookup_family: V4_ONLY
    transport_socket:
      name: envoy.transport_sockets.tls
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: login.microsoftonline.com
    load_assignment:
      cluster_name: entra_token
      endpoints:
      - lb_endpoints:
        - endpoint:
            address:
              socket_address: { address: login.microsoftonline.com, port_value: 443 }
EOF

# Recreate the ConfigMap so re-runs pick up the new config.
kubectl create configmap envoy-config -n echo \
  --from-file=envoy.yaml=/tmp/envoy.yaml \
  --dry-run=client -o yaml | kubectl apply -f -
```

```bash
# SDS - loads the secrets as token/hmac for the oauth2 filter
cat > /tmp/token.yaml <<'EOF'
resources:
- "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.Secret
  name: token
  generic_secret:
    secret: { filename: "/etc/envoy/secret/client-secret" }
EOF

cat > /tmp/hmac.yaml <<'EOF'
resources:
- "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.Secret
  name: hmac
  generic_secret:
    secret: { filename: "/etc/envoy/secret/hmac-secret" }
EOF

kubectl create configmap envoy-sds -n echo \
  --from-file=token.yaml=/tmp/token.yaml \
  --from-file=hmac.yaml=/tmp/hmac.yaml \
  --dry-run=client -o yaml | kubectl apply -f -
```

---

## 8. Envoy Deployment + Service (ClusterIP)

```bash
cat > /tmp/envoy-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: envoy
  namespace: echo
spec:
  replicas: 1
  selector: { matchLabels: { app: envoy } }
  template:
    metadata: { labels: { app: envoy } }
    spec:
      containers:
      - name: envoy
        image: envoyproxy/envoy:v1.31-latest
        args: ["-c", "/etc/envoy/envoy.yaml", "-l", "info"]
        ports: [{ containerPort: 10000 }]
        volumeMounts:
        - { name: config, mountPath: /etc/envoy/envoy.yaml, subPath: envoy.yaml }
        - { name: sds, mountPath: /etc/envoy/sds }
        - { name: secret, mountPath: /etc/envoy/secret }
      volumes:
      - { name: config, configMap: { name: envoy-config } }
      - { name: sds, configMap: { name: envoy-sds } }
      - { name: secret, secret: { secretName: envoy-oauth } }
---
apiVersion: v1
kind: Service
metadata:
  name: envoy
  namespace: echo
spec:
  type: ClusterIP
  ports: [{ port: 80, targetPort: 10000 }]
  selector: { app: envoy }
EOF

kubectl apply -f /tmp/envoy-deploy.yaml

# If the config/secret changed on a re-run, force a fresh rollout.
kubectl rollout restart deploy/envoy -n echo
kubectl rollout status deploy/envoy -n echo

# Pod should be Running 1/1 and the Service should have an endpoint.
kubectl get pods -n echo -l app=envoy
kubectl get endpoints envoy -n echo
```

---

## 9. ClusterIssuer — Let's Encrypt (staging)

> Use staging first to avoid exhausting Let's Encrypt rate limits if anything fails.

```bash
cat > /tmp/issuer-staging.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: ${ACME_EMAIL}
    privateKeySecretRef: { name: letsencrypt-staging-key }
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
EOF

kubectl apply -f /tmp/issuer-staging.yaml
```

---

## 10. Ingress with TLS

```bash
cat > /tmp/ingress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo-tls
  namespace: echo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
    nginx.ingress.kubernetes.io/proxy-buffer-size: "16k"   # large OAuth2 cookie
spec:
  ingressClassName: nginx
  tls:
  - hosts: ["${NIP_HOST}"]
    secretName: echo-tls-cert
  rules:
  - host: ${NIP_HOST}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: envoy
            port: { number: 80 }
EOF

kubectl apply -f /tmp/ingress.yaml

# Wait for the cert (staging) - READY should become True
kubectl get certificate -n echo -w
```

---

## 11. Switch to production Let's Encrypt (trusted cert)

```bash
cat > /tmp/issuer-prod.yaml <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ${ACME_EMAIL}
    privateKeySecretRef: { name: letsencrypt-prod-key }
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
EOF
kubectl apply -f /tmp/issuer-prod.yaml

# Repoint the Ingress to prod and force re-issuance
kubectl annotate ingress echo-tls -n echo \
  cert-manager.io/cluster-issuer=letsencrypt-prod --overwrite
kubectl delete secret echo-tls-cert -n echo

kubectl get certificate -n echo -w     # wait for READY=True
```

---

## 12. Test

```bash
kubectl get pods -n echo
echo "Open in a browser: https://$NIP_HOST/"

# Expected: 302 -> login.microsoftonline.com
curl -I https://$NIP_HOST/
```

Browser flow:
`https://<IP>.nip.io/` → redirect to the Entra ID login → after signing in, the
content from echo-server is shown.

---

## Troubleshooting

```bash
kubectl get pods -n echo
kubectl logs -n echo deploy/envoy --tail=50
kubectl get certificate -n echo
kubectl describe certificate echo-tls-cert -n echo
kubectl get challenges -A
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```

Inspect OAuth2 flow details (useful when debugging the callback):

```bash
kubectl patch deploy/envoy -n echo --type=json -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/args",
   "value":["-c","/etc/envoy/envoy.yaml","-l","info","--component-log-level","oauth2:debug,http:debug,upstream:debug"]}
]'
kubectl rollout status deploy/envoy -n echo
kubectl logs -n echo deploy/envoy -f --tail=0
```

### Issues encountered and their fixes

| Symptom | Cause | Fix |
|---|---|---|
| `az` commands fail with `TokenIssuedBeforeRevocationTimestamp` / `InteractionRequired`; `APP_ID` ends up empty | Azure CLI token revoked mid-session by Continuous Access Evaluation (CAE) | Re-run `az login --tenant $TENANT_ID`, then re-run step 5. Step 5 now re-logs in and guards `APP_ID` before resetting the secret. |
| Envoy `CrashLoopBackOff`: `OAuth2CredentialsValidationError.ClientId: value length must be at least 1 characters` | `$APP_ID` was empty when the config heredoc ran (lost shell variable) | Re-export `APP_ID` (step 5), regenerate the ConfigMap, `kubectl rollout restart deploy/envoy`. Guards in step 7 abort early now. |
| Envoy fails to start: `GenericSecretSdsApi: node 'id' and 'cluster' are required` | Missing `node` section when using file-based SDS | Added `node.id` and `node.cluster` |
| `503` from nginx | Envoy didn't start → no Service endpoint | Fix the Envoy config, restart Envoy |
| After signing in, "OAuth flow failed", no `AADSTS` in the log | `login.microsoftonline.com` resolved to **IPv6**, cluster has no IPv6 routing → `Network is unreachable` | `dns_lookup_family: V4_ONLY` on the `entra_token` cluster |
| Possible `401` from the token endpoint | Secret sent as Basic auth instead of in the body | `auth_type: URL_ENCODED_BODY` |
| Cookie-signing problems | `hmac-secret` as base64 instead of raw bytes | `openssl rand -hex 32` |

---

## Cleanup (rollback)

```bash
# Delete the whole cluster along with all resources in it
az aks delete --resource-group "$RG" --name "$AKS" --yes --no-wait

# Delete the Resource Group (if nothing else lives in it)
az group delete --name "$RG" --yes --no-wait

# Delete the App Registration
az ad app delete --id "$APP_ID"
```
