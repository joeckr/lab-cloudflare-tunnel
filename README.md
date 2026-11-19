# lab-cloudflare-tunnel

A reusable, hardened Helm chart for deploying a **remotely managed Cloudflare Tunnel (`cloudflared`)** in a homelab Kubernetes cluster enforcing `restricted` Pod Security Standards (PSS), designed to route external traffic into an in-cluster **Traefik Kubernetes Gateway**.

---

## Architecture

```text
Internet ──> Cloudflare Edge ──(Tunnel)──> cloudflared Pod ──> Traefik Gateway Service ──(HTTPRoute)──> App Pods
```

- **Remotely Managed Tunnel**: Routing rules and public hostnames are managed in the **Cloudflare Zero Trust Dashboard** rather than local ConfigMaps. The in-cluster `cloudflared` pod reads its `TUNNEL_TOKEN` directly from a pre-existing native Kubernetes `Secret`.
- **Traefik Gateway Integration**: Point Cloudflare Tunnel public hostnames at your Traefik Gateway's internal ClusterIP Service (e.g., `http://gateway-traefik.gateway.svc.cluster.local:80`). `cloudflared` preserves the `Host` header so Traefik `Gateway` and `HTTPRoute` resources route requests to your backend services as usual.
- **No Custom Image Required**: The upstream `cloudflare/cloudflared` image is built on `distroless/nonroot` (`USER 65532:65532`), requires no filesystem writes in remote token mode, and binds only unprivileged port `2000` for `/ready` metrics.

---

## Security & PSS `restricted` Compliance

This chart is configured to pass Kubernetes **Pod Security Admission (PSA)** `restricted` enforcement out of the box:

| Security Dimension | Restricted Standard Requirement | Chart Implementation |
|---|---|---|
| **Non-Root User** | `runAsNonRoot: true`, non-zero UID | `runAsNonRoot: true`, `runAsUser: 65532` (matches upstream distroless `nonroot`) |
| **Root Filesystem** | Read-only recommended / supported | `readOnlyRootFilesystem: true` |
| **Capabilities** | Drop `ALL` capabilities | `capabilities.drop: ["ALL"]` |
| **Privilege Escalation** | `allowPrivilegeEscalation: false` | Enforced in container `securityContext` |
| **Seccomp Profile** | `RuntimeDefault` or `Localhost` | `seccompProfile.type: RuntimeDefault` |
| **Service Account Token** | Disabled when Kubernetes API access isn't needed | `automountServiceAccountToken: false` on both Pod and ServiceAccount |
| **Secret Handling** | No plaintext credentials in Helm values | References a pre-existing native Kubernetes `Secret` only |
| **Ports** | Unprivileged ports (> 1024) | Outbound-only to Cloudflare edge; metrics/liveness on port `2000` |
| **Control-Plane Scheduling** | Optional lab cluster scheduling | Includes `control-plane` and `master` `NoSchedule` tolerations |

---

## Prerequisites

1. **Cloudflare Zero Trust Tunnel**:
   - In the Cloudflare Zero Trust Dashboard, go to **Networks > Tunnels** and create a **Cloudflared** tunnel.
   - Copy the **Tunnel Token** (starts with `eyJ...`).
2. **Kubernetes Namespace (`cloudflared`)**:
   - Create the `cloudflared` namespace and enforce `restricted` PSS:
     ```sh
     kubectl create namespace cloudflared
     kubectl label namespace cloudflared \
       pod-security.kubernetes.io/enforce=restricted \
       pod-security.kubernetes.io/enforce-version=latest
     ```
3. **Native Kubernetes Secret (`tunnel-token`)**:
   - Create the `tunnel-token` Secret in the `cloudflared` namespace (the chart defaults to `secretName: "tunnel-token"` and `secretKey: "token"`):
     ```sh
     kubectl -n cloudflared create secret generic tunnel-token \
       --from-literal=token="<YOUR_CLOUDFLARE_TUNNEL_TOKEN>"
     ```
   > If your existing `tunnel-token` Secret uses a different data key, override it at install time with `--set cloudflare.secretKey=<YOUR_KEY>`.
4. **Traefik Gateway Service**:
   - Note the internal Kubernetes Service DNS name and port for your Traefik Gateway (for `lab-traefik-kube-gateway`, this is `gateway-traefik.gateway.svc.cluster.local:80`, which forwards to Traefik's `web` listener on container port `8000`).

---

## Deploying to Your Cluster

### Installing from the Local Chart

Because `values.yaml` defaults to `cloudflare.secretName: "tunnel-token"`, you can install directly into the `cloudflared` namespace:

```sh
helm install cloudflare-tunnel ./chart \
  --namespace cloudflared
```

If you need to specify a different Secret name or key:

```sh
helm install cloudflare-tunnel ./chart \
  --namespace cloudflared \
  --set cloudflare.secretName=tunnel-token \
  --set cloudflare.secretKey=token
```

### Installing from GHCR (After Release)

Once published by the `release.yml` workflow, you can install directly from OCI:

```sh
helm install cloudflare-tunnel oci://ghcr.io/joeckr/charts/cloudflare-tunnel \
  --namespace cloudflared
```

---

## Routing Traffic to Traefik Gateway

In the Cloudflare Zero Trust Dashboard under your tunnel's **Public Hostname** tab:

1. **Public Hostname**: Add your domain or subdomain (e.g., `app.example.com` or `*.example.com`).
2. **Service Type**: `HTTP` (or `HTTPS` if terminating TLS at Traefik).
3. **URL**: Point to Traefik's internal cluster Service, e.g.:
   ```text
   gateway-traefik.gateway.svc.cluster.local:80
   ```

> [!TIP]
> **HTTPS Redirects & Client IPs:**
> - Because Cloudflare terminates TLS at the edge, forwarding `HTTP` to Traefik's `web` Service port (`:80` -> container port `:8000`) works without Origin certificates. Ensure Traefik does not force an HTTP-to-HTTPS redirect on that entrypoint, or forward `HTTPS` with **No TLS Verify** enabled.
> - Cloudflare sends the original client IP in `CF-Connecting-IP` and `X-Forwarded-For`. To preserve `X-Forwarded-For` in Traefik, configure Traefik's entrypoint `forwardedHeaders.trustedIPs` to trust your cluster's Pod CIDR.

---

## Verification & Troubleshooting

Check that the pod passes PSA admission and connects to Cloudflare's edge:

```sh
# Check pod status (should become Running and pass /ready liveness probe)
kubectl -n cloudflared get pods -l app.kubernetes.io/name=cloudflare-tunnel

# Inspect events if the pod fails to schedule or start
kubectl -n cloudflared describe pod -l app.kubernetes.io/name=cloudflare-tunnel

# Stream cloudflared connection logs
kubectl -n cloudflared logs -l app.kubernetes.io/name=cloudflare-tunnel -f
```

You can also inspect the local `/ready` and Prometheus metrics endpoint via port-forwarding:

```sh
kubectl -n cloudflared port-forward deploy/cloudflare-tunnel 2000:2000
curl http://localhost:2000/ready
```

---

## Helm Chart Values

| Key | Default | Description |
|---|---|---|
| `cloudflare.secretName` | `"tunnel-token"` | Required name of the pre-existing Kubernetes Secret containing the tunnel token |
| `cloudflare.secretKey` | `"token"` | Key inside the Secret containing the tunnel token |
| `image.repository` | `"cloudflare/cloudflared"` | Container image repository |
| `image.tag` | `"latest"` | Container image tag |
| `image.pullPolicy` | `"IfNotPresent"` | Image pull policy |
| `replicaCount` | `1` | Number of `cloudflared` replicas |
| `serviceAccount.create` | `true` | Create a dedicated ServiceAccount |
| `serviceAccount.automount` | `false` | Mount Kubernetes API token into the pod |
| `metrics.port` | `2000` | Port for `/ready` liveness probe and Prometheus metrics |
| `podSecurityContext` | `runAsNonRoot: true`, `runAsUser: 65532`, `RuntimeDefault` | Pod-level security context |
| `securityContext` | `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, drop `ALL` | Container-level security context |
| `resources` | `requests: 50m/64Mi`, `limits: 250m/256Mi` | CPU and memory requests/limits |
| `tolerations` | `control-plane` & `master` `NoSchedule` | Allows scheduling on single-node / control-plane lab nodes |

---

## Local Testing & Development

### Setup Tooling

```sh
mise run install
```

### Local Compose Test (`compose.yml`)

```sh
export TUNNEL_TOKEN="<YOUR_CLOUDFLARE_TUNNEL_TOKEN>"
mise run compose
mise run logs
mise run down
```

### Helm Lint & Template Rendering

```sh
mise run helm-l
mise run helm-t
```

### Available `mise` Tasks

| Task | Description | Command |
|---|---|---|
| `install` | Install tools and set up git hooks | `hk install --mise` |
| `hk` (`check`) | Run all linters and hook checks | `hk check --all` |
| `compose` | Start local `cloudflared` container | `podman compose up -d` |
| `down` | Stop local Compose stack | `podman compose down` |
| `logs` | View local Compose logs | `podman compose logs -f` |
| `play` | Test rendered manifests with `podman play kube` | `podman play kube rendered.yaml --publish-all` |
| `play-d` | Stop `podman play kube` pods | `podman play kube rendered.yaml --down` |
| `helm-l` | Lint Helm chart | `helm lint chart/` |
| `helm-t` | Render Helm chart templates to `rendered.yaml` | `helm template test chart/ > rendered.yaml` |
| `helm-i` | Install test release to current cluster | `helm install test chart/` |
| `helm-u` | Uninstall test release from cluster | `helm uninstall test` |
| `trivy-fs` | Scan repository for vulnerabilities | `trivy fs .` |

---

## Repository Structure

```text
.
├── .github/workflows/
│   ├── lint.yml                 # actionlint, commitlint, shellcheck
│   ├── security.yml             # betterleaks and zizmor audits
│   ├── release.yml              # Semantic versioning & Helm OCI push to GHCR
│   └── test_release.yml         # PR dry-run for release & chart packaging
├── chart/
│   ├── Chart.yaml               # Helm chart metadata
│   ├── values.yaml              # Default configuration values
│   ├── .helmignore              # Chart packaging ignore rules
│   └── templates/
│       ├── _helpers.tpl         # Name, label, and Secret resolution helpers
│       ├── deployment.yaml      # Hardened cloudflared Deployment
│       └── serviceaccount.yaml  # Unprivileged ServiceAccount
├── compose.yml                  # Local Podman Compose definition
├── hk.pkl                       # hk git hooks and lint checks
├── mise.toml                    # Mise tools and task runner
└── README.md
```

---

## Conventional Commits & Releases

Commits must follow [Conventional Commits](https://www.conventionalcommits.org/):
- `feat:` -> Minor release (`v0.1.0` -> `v0.2.0`)
- `fix:` / `perf:` -> Patch release (`v0.1.0` -> `v0.1.1`)
- `feat!:` / `BREAKING CHANGE` -> Major release
- `chore:`, `docs:`, `ci:`, `test:`, `refactor:` -> No release bump

On merge to `main`, `.github/workflows/release.yml` automatically calculates the next version, updates `chart/Chart.yaml`, creates the Git tag and GitHub Release, and pushes the Helm chart to GHCR.

## License

Please refer to the [LICENSE](LICENSE) file for details.
