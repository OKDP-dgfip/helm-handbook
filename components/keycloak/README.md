# Keycloak

## Purpose & Scope

Deploy a minimal Keycloak server (IAM / SSO) on a Kubernetes cluster.

Suitable for: testing, demos. 
***Not suitable for: production*** (no persistent database, plaintext credentials, no TLS, no exposure).

## Deployment

The release name `keycloak` is used as a prefix for the created resources.

### Configuration

| Parameter | Effect |
|---|---|
| `fullnameOverride` | Overrides the default fullname (default: `keycloak-keycloakx`, `keycloak` is set in the values file) |
| `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD` | Creates the initial admin account (`admin` / `admin`) |
| `kc.sh start` | Keycloak "production" mode (not `start-dev`) |
| `--http-port=8080` | HTTP listening port |
| `--hostname-strict=false` | Hostname is derived from the request (handy for testing, ***discouraged in production***) |
| `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD` | Creates the initial admin account (`admin` / `admin`) |

- [Default values file](https://github.com/codecentric/helm-charts/blob/master/charts/keycloakx/values.yaml)

### Installation

```bash
helm dependency build .
helm install keycloak . -n keycloak --create-namespace
```

### Ressources

| Resource | Name (approx.) | Role |
|---|---|---|
| StatefulSet | `keycloak` | 1 Keycloak replica |
| ClusterIP Service | `keycloak-http` | Port 80 → 8080 |
| Headless Service | `keycloak-headless` | Pod discovery for clustering |
| ServiceAccount | `keycloak` | Pod identity |
| Helm release Secret | `sh.helm.release.v1.keycloak.v1` | Release history and values (created by Helm) |

**Not created by default**: Ingress, PVC, NetworkPolicy, ServiceMonitor, HPA, PodDisruptionBudget, application Secret.

### Images

| Image | Usage |
|---|---|
| `quay.io/keycloak/keycloak:<tag>` | Keycloak server. The tag follows the chart's `appVersion`. |

The `keycloakx` chart ships **no database sub-chart** (unlike the older `keycloak` chart).

### Storage

- **No PVC, no persistent volume.**
- With no database configured, Keycloak uses an **embedded H2** database in file mode, at `/opt/keycloak/data/h2/`, i.e. in the container's writable layer.
- When the container is recreated (pod deleted, redeployment, crash followed by a fresh container), **all data is lost** (realms, users, clients, sessions).

Check:
```bash
kubectl exec -n keycloak keycloak-0 -- ls /opt/keycloak/data/h2
# keycloakdb.mv.db
# keycloakdb.trace.db
```

> [!NOTE]
> If the H2 database was lost during the restart, Keycloak started up with an empty database. It then read KEYCLOAK_ADMIN=admin and KEYCLOAK_ADMIN_PASSWORD=admin again, and recreated the account. This is why you will be able to log in with admin/admin.

### Accessing Keycloak

**Access from a workstation:**
```bash
kubectl port-forward -n keycloak svc/keycloak-http 8080:80
```
Then open `http://localhost:8080` (the path might change depending on the chart version or `http.relativePath` setting).

### Teardown

```bash
helm uninstall keycloak -n keycloak
kubectl delete namespace keycloak
```

## References

- [Keycloakx Helm Chart](https://github.com/codecentric/helm-charts/tree/master/charts/keycloakx)
