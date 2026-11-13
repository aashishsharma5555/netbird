# NetBird on AKS: Deployment and Test Log

Last updated: 2026-11-13 (Asia/Kolkata)

This document records the NetBird AKS work, the commands used or needed to
repeat the checks, the outcomes, and the current state. Commands are written
for Windows PowerShell from the `netbird-k8s` directory unless noted.

This is a consolidated runbook, not a byte-for-byte terminal transcript. The
early bootstrap/Gateway/Secret shell history was summarized before this
documentation request, so those original one-off command lines were not
retained. Repeatable inspection, Helm, rollout, and test commands are recorded
below; secret creation values are intentionally not reconstructed.

> **Sensitive values omitted:** Kubernetes Secret data, generated passwords,
> encryption keys, and relay credentials are intentionally not documented.
> Do not print or commit Secret values. This is a temporary HTTP-only test
> deployment, not a production-ready configuration.

## Current status

| Item | Current state |
|---|---|
| AKS context | `netbird-cloud4green-aks` |
| Namespace / Helm release | `netbird` / `netbird` |
| Helm chart | Bundled `netbird-1.9.0` |
| Gateway | `netbird-gateway`, class `approuting-istio`, `PROGRAMMED=True` |
| Public address | `20.242.192.58` (HTTP only) |
| Dashboard | 2/2 ready; one pod on each `agentpool2` node |
| Dashboard HPA | CPU target 70%; min=2, max=10 |
| Management / relay / signal | One replica each; unchanged by the node-spread request |
| Public dashboard / OIDC discovery | HTTP 200 / HTTP 200 |
| Dashboard authentication | `USE_AUTH0=false` and `api` scope deployed; fresh-browser login retest pending |

Verified dashboard placement:

- `aks-agentpool2-14087607-vmss000008`
- `aks-agentpool2-14087607-vmss000009`

The two `agentpool` nodes are not the `agentpool2` nodes. The AKS-managed
Application Routing Gateway pods remain managed separately from the NetBird
dashboard scheduling rules.

## Work timeline

### 1. Initial inspection and AKS access

- Read the repository, bundled Helm chart, values, templates, and Kubernetes
  snapshots/overlays.
- The repository's `backend.split.*` values did not match the bundled chart's
  `management`, `signal`, `relay`, and `dashboard` values.
- Azure access initially required re-authentication due to an MFA policy.
- The user re-authenticated and enabled Application Routing.
- Confirmed the cluster had the `approuting-istio` GatewayClass and Gateway API
  route support. The Gateway was created before the NetBird release; it initially
  returned 404 because no NetBird workloads/routes existed yet.

Useful inspection commands:

```powershell
az account show --output table
az aks show --resource-group netbird-cloud4green-aks_group `
  --name netbird-cloud4green-aks --output table
kubectl config current-context
kubectl get nodes -o wide
kubectl get gatewayclass
kubectl get crd | Select-String 'gateway|httproute|grpcroute'
helm list --all-namespaces
```

If the AKS context is not already configured, the usual credential command is:

```powershell
az aks get-credentials --resource-group netbird-cloud4green-aks_group `
  --name netbird-cloud4green-aks
```

The namespace/Gateway were created during the session, but their temporary
manifest/inline command was not preserved in the available command history.
Their current state can be checked with the commands above and the live-status
commands at the end of this document.

The user enabled Application Routing in Azure; this document does not claim
that action was performed by a CLI command.

### 2. Gateway and first NetBird installation

- Created namespace `netbird` and HTTP Gateway `netbird-gateway`.
- Gateway received public IP `20.242.192.58`.
- The first chart install used the chart's default NetBird `0.46.0`; management
  did not support the embedded identity-provider configuration and crashed.
- Updated the test values to management/signal/relay `0.82.1` and dashboard
  `v2.96.1`, and reinstalled/upgraded the release.
- The final test configuration uses embedded local identity, SQLite-backed
  management persistence, and HTTP Gateway API routes.
- The management trusted proxy CIDR was restricted to the AKS pod CIDR
  `10.244.0.0/16`.
- Created Kubernetes Secret `netbird-secrets`. Its contents are intentionally
  excluded from this log.

The active Helm input is [aks-test-values.yaml](./aks-test-values.yaml).
The Gateway routes are rendered from that file. Do not apply old
`current-deployment.yaml` snapshots as a replacement for the active Helm
release.

Deployment/verification command sequence:

```powershell
Set-Location 'C:\Users\User\Downloads\reyvanx\netbird\netbird-k8s'
helm lint .\netbird -f .\aks-test-values.yaml
helm template netbird .\netbird -n netbird -f .\aks-test-values.yaml
helm upgrade --install netbird .\netbird -n netbird `
  --create-namespace -f .\aks-test-values.yaml --wait --timeout 5m
helm status netbird -n netbird
kubectl get gateway netbird-gateway -n netbird -o wide
kubectl get httproute,grpcroute -n netbird
kubectl get deployments,pods,services -n netbird -o wide
curl.exe -s -o NUL -w '%{http_code}\n' --max-time 15 http://20.242.192.58/
curl.exe -s -o NUL -w '%{http_code}\n' --max-time 15 `
  http://20.242.192.58/oauth2/.well-known/openid-configuration
```

Observed successful checks: release `deployed`, Gateway programmed, workloads
ready, dashboard HTTP 200, and OIDC discovery HTTP 200.

### 3. LoadBalancer and autoscaling/auto-healing tests

Initial live checks found:

- Gateway LoadBalancer address `20.242.192.58`, HTTP port 80.
- The repository had CPU 70% HPA manifests, but they were not initially active
  for NetBird. Dashboard also had no CPU request, which is required for CPU
  utilization-based HPA calculations.
- Management uses SQLite on a single `ReadWriteOnce` volume. Scaling management
  to multiple replicas without changing its storage design is unsafe; its HPA
  was deliberately not applied.

Applied the dashboard-only overlays:

```powershell
kubectl apply -f .\k8s\dashboard\resources.yaml
kubectl apply -f .\k8s\dashboard\hpa.yaml
kubectl rollout status deployment/netbird-dashboard -n netbird --timeout=120s
kubectl get hpa netbird-dashboard-hpa -n netbird -o wide
kubectl top pods -n netbird --containers
```

The dashboard resource overlay sets requests to `100m` CPU / `128Mi` memory
and limits to `500m` CPU / `512Mi` memory.

For the auto-healing check, one ready dashboard pod was deleted and the
Deployment recreated it:

```powershell
kubectl get pods -n netbird -l app.kubernetes.io/name=netbird-dashboard -o wide
kubectl delete pod <one-ready-dashboard-pod> -n netbird --wait=true --timeout=60s
kubectl wait --for=condition=available deployment/netbird-dashboard `
  -n netbird --timeout=180s
kubectl get deployment netbird-dashboard -n netbird -o wide
kubectl get pods -n netbird -l app.kubernetes.io/name=netbird-dashboard -o wide
curl.exe -s -o NUL -w '%{http_code}\n' --max-time 15 http://20.242.192.58/
```

Observed result: deployment returned to 2/2 available and public HTTP remained
200.

For autoscaling, a bounded CPU load was run in one dashboard pod; its exact pod
name was `netbird-dashboard-d57fd4d69-trk8k` at the time:

```powershell
kubectl exec netbird-dashboard-d57fd4d69-trk8k -n netbird -- `
  sh -c 'timeout 90s yes >/dev/null'
```

The remote exec ended with exit code 143. During the test, HPA CPU utilization
reached 113% against the 70% target and HPA selected 4 replicas, then briefly
selected 8 while the metrics/load settled. It did not exceed the then-current
max of 10. After load stopped, CPU returned to about 1% and replicas scaled
back down to 2 after the HPA stabilization/scale-down interval. No temporary
load-test Kubernetes resource was created.

Relevant observation commands:

```powershell
kubectl get hpa netbird-dashboard-hpa -n netbird -o wide
kubectl describe hpa netbird-dashboard-hpa -n netbird
kubectl get deployment netbird-dashboard -n netbird -o wide
kubectl top pods -n netbird --containers
```

### 4. Dashboard fixed at two replicas and distributed across `agentpool2`

The user requested that the app run on the two named `agentpool2` nodes and
that min/max match the desired pod count. After confirmation, the chosen scope
was dashboard only: min=max=2; management stays at one replica pending a safe
HA storage design.

Updated files:

- [aks-test-values.yaml](./aks-test-values.yaml): dashboard `replicaCount: 2`,
  existing resource request/limit values, node selector
    `kubernetes.azure.com/agentpool: agentpool2`, and a topology spread
    constraint on `kubernetes.io/hostname` with `maxSkew: 1` so up to 10
    replicas can be balanced across the two eligible nodes.
  - [netbird/templates/dashboard-deployment.yaml](./netbird/templates/dashboard-deployment.yaml):
    set rolling updates to `maxSurge: 0`, `maxUnavailable: 1` so old pods with
    their prior required anti-affinity can be replaced without a scheduling
    deadlock, and render the dashboard topology spread constraint from values.
- [k8s/dashboard/hpa.yaml](./k8s/dashboard/hpa.yaml): `minReplicas: 2` and
  `maxReplicas: 10` (CPU utilization target remains 70%).

Validation and deployment commands:

```powershell
helm lint .\netbird -f .\aks-test-values.yaml
helm template netbird .\netbird -n netbird -f .\aks-test-values.yaml
kubectl apply -f .\k8s\dashboard\hpa.yaml
helm upgrade netbird .\netbird -n netbird -f .\aks-test-values.yaml `
  --wait --timeout 5m
kubectl rollout status deployment/netbird-dashboard -n netbird --timeout=180s
kubectl get pods -n netbird -l app.kubernetes.io/name=netbird-dashboard -o wide
kubectl get hpa netbird-dashboard-hpa -n netbird -o wide
kubectl get deployments -n netbird -o wide
```

Verified at the time: exactly two ready dashboard pods, one on each named
node; HPA min/max both 2; public dashboard HTTP 200. The HPA's latest measured
CPU was 1% against target 70%. The max was subsequently changed to 10 at the
user's request; the dashboard HPA can now scale from 2 through 10 replicas.
The initial required anti-affinity rule blocked a rolling upgrade and would
have prevented more than two pods on the selected two-node pool. Soft
anti-affinity then did not guarantee placement across both nodes; the final
configuration uses a hard topology spread constraint instead, which permits
multiple replicas per node while enforcing balanced placement.

### 5. Screenshot authentication diagnosis and configuration update

The screenshot shows that the dashboard HTML loads, then displays
`Error: Unauthenticated`. The Gateway and workloads were healthy. Initial
read-only checks returned:

- `GET /` -> HTTP 200
- OIDC discovery -> HTTP 200; issuer `http://20.242.192.58/oauth2`
- OIDC authorization endpoint -> HTTP 302 to the local sign-in flow
- Unauthenticated management API request (`/api/peers`) -> HTTP 401, as expected
  when no valid token is supplied
- Dashboard logs included `USE_AUTH0 environment variable must be set`
- Running dashboard environment did not contain `USE_AUTH0`
- Configured `AUTH_SUPPORTED_SCOPES` did not contain `api`
- Even after those variables were corrected, the browser still failed. Recent
  Dashboard logs showed the IdP returning to `/nb-auth?code=...`, followed by
  HTTP 404 from Dashboard. Direct checks confirmed `/nb-auth` and
  `/nb-silent-auth` were 404 while `/` was 200.

Commands used for diagnosis:

```powershell
kubectl logs <dashboard-pod> -n netbird --since=30m --all-containers=true
kubectl logs <management-pod> -n netbird --since=30m --all-containers=true
kubectl exec <dashboard-pod> -n netbird -- env
curl.exe -s -o NUL -w '%{http_code}\n' --max-time 15 http://20.242.192.58/
curl.exe -s -o NUL -w '%{http_code}\n' --max-time 15 `
  http://20.242.192.58/oauth2/.well-known/openid-configuration
curl.exe -s -o NUL -w '%{http_code}\n' --max-time 15 `
  http://20.242.192.58/api/peers
curl.exe -s -o NUL -w '%{http_code}\n' --max-time 15 http://20.242.192.58/nb-auth
curl.exe -sS -D - -o NUL --max-time 15 `
  'http://20.242.192.58/oauth2/auth?client_id=netbird-dashboard&redirect_uri=http%3A%2F%2F20.242.192.58%2Fnb-auth&response_type=code&scope=openid%20profile%20email&state=diagnostic&nonce=diagnostic'
```

The actual callback failure was a version/config mismatch. The current guide
uses `/nb-auth` and `/nb-silent-auth`, but the pinned Dashboard
`v2.96.1` image serves static Next.js routes and returns 404 for those paths.
Its versioned source defaults to `/#callback` and `/#silent-callback`
([config.ts](https://github.com/netbirdio/dashboard/blob/v2.96.1/src/utils/config.ts));
the browser sends the callback request to `/` because URL fragments are not
sent to the web server. The first `USE_AUTH0`/scope corrections were valid but
did not address this callback route mismatch.

Fixed by aligning both the Dashboard callback paths and the embedded IdP's
registered redirect URIs to the pinned image:

- Dashboard `AUTH_REDIRECT_URI=/#callback`
- Dashboard `AUTH_SILENT_REDIRECT_URI=/#silent-callback`
- Management `DashboardRedirectURIs` set to the absolute `/#callback` and
  `/#silent-callback` URLs
- Dashboard retains `USE_AUTH0=false` and the `api` scope

The Management ConfigMap change triggered a rollout while Management's SQLite
PVC was mounted by a single pod. After the failed default surge attempt, the
existing PVC was kept Bound, Management was briefly scaled to zero to release
it, and Helm recreated one Management pod with a no-surge rollout strategy
(`maxSurge: 0`, `maxUnavailable: 1`). No user/account database or Secret was
reset.

Applied and verified with:

```powershell
helm lint .\netbird -f .\aks-test-values.yaml
helm template netbird .\netbird -n netbird -f .\aks-test-values.yaml
helm upgrade netbird .\netbird -n netbird -f .\aks-test-values.yaml `
  --wait --timeout 5m
kubectl rollout status deployment/netbird-management -n netbird --timeout=180s
kubectl rollout status deployment/netbird-dashboard -n netbird --timeout=180s
kubectl get pods -n netbird -l app.kubernetes.io/name=netbird-dashboard -o wide
kubectl get hpa netbird-dashboard-hpa -n netbird -o wide
curl.exe -s -o NUL -w '%{http_code}\n' --max-time 15 http://20.242.192.58/
curl.exe -s -o NUL -w '%{http_code}\n' --max-time 15 `
  http://20.242.192.58/oauth2/.well-known/openid-configuration
```

The official [Self-Hosting Quickstart](https://docs.netbird.io/selfhosted/selfhosted-quickstart)
was reviewed. Its `getting-started.sh` installs a Docker Compose stack on a
public Linux VM, including its own reverse-proxy choices; it is not a
Kubernetes/Helm installer and was not run against this existing AKS cluster.
This deployment already uses Management's embedded IdP, and its existing
owner/account and SQLite data were retained. No Keycloak or Entra migration
was performed.

The `Not secure` browser indicator is because the test endpoint uses HTTP;
production should use HTTPS and a real hostname/certificate. After applying
the callback fix, the Dashboard root and OIDC discovery returned HTTP 200; an
authorization request using `/#callback` returned HTTP 302 to local sign-in;
and the unauthenticated management API returned HTTP 401 as expected. The
previous `/nb-auth` path is no longer configured. These checks do not replace
a full interactive browser login test, so verify using a fresh/private browser
session.

The final auth and scheduling configuration was applied with Helm revision 10
on 2026-11-13. Dashboard pods were confirmed on separate `agentpool2` nodes;
HPA min/max are 2/10 with a 70% CPU target. The Dashboard uses a hard topology
spread constraint to balance pods. The Management Deployment uses a
`maxSurge: 0`, `maxUnavailable: 1` strategy to avoid overlapping mounts of
its single-writer SQLite PVC.

## Current live verification commands

Run from any directory with the correct `kubectl` context:

```powershell
kubectl config current-context
helm status netbird -n netbird
kubectl get gateway netbird-gateway -n netbird -o wide
kubectl get deployments -n netbird -o wide
kubectl get pods -n netbird -o wide
kubectl get hpa -n netbird -o wide
kubectl top pods -n netbird --containers
curl.exe -s -o NUL -w '%{http_code}\n' --max-time 15 http://20.242.192.58/
curl.exe -s -o NUL -w '%{http_code}\n' --max-time 15 `
  http://20.242.192.58/oauth2/.well-known/openid-configuration
```

## Files and configurations not applied

- `k8s/*/current-deployment.yaml` files are old exported snapshots and were not
  reapplied over the Helm release.
- Management, relay, and signal HPA/resource overlays were not applied.
- `keycloak.yaml` is a separate Keycloak deployment manifest. It was inspected
  as supplied but was not used for this NetBird deployment.
- Old manifest values containing stale IPs or plaintext credentials must not
  be copied into a new deployment.

## Important limitations

1. The endpoint is HTTP-only and uses a public IP; do not treat it as
   production-ready.
2. The dashboard is fixed at exactly two replicas. To restore autoscaling,
   intentionally set an appropriate `maxReplicas` greater than 2 and verify
   node capacity/cost first.
3. Do not scale management beyond one replica while it uses SQLite on the
   current single-writer persistent volume.
4. `aks-test-values.yaml` contains deployment configuration, not Secret
   contents. Secret values remain in Kubernetes and must not be added here.
