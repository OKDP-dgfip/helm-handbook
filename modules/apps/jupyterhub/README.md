[![Helm](https://img.shields.io/badge/helm-3+-blue.svg)](https://helm.sh/)
[![Chart](https://img.shields.io/badge/chart-jupyterhub%204.3.3-orange.svg)](https://hub.jupyter.org/helm-chart/)
[![JupyterHub](https://img.shields.io/badge/jupyterhub-5.4-orange.svg)](https://jupyterhub.readthedocs.io/)
[![License Apache2](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0)

# JupyterHub

JupyterHub on Kubernetes with the OKDP notebook image, installed with the minimum of prerequisites: one Helm release, one values file, and a cluster that only provides a default StorageClass. Each user signs in, gets a personal JupyterLab server in its own pod, and keeps the files in a personal volume.

## What it installs

| Component | Kind | Role |
|---|---|---|
| `hub` | Deployment | Signs users in, starts and stops their servers through the Kubernetes API, keeps its state in a sqlite database on the volume `hub-db-dir`. |
| `proxy` | Deployment | Single entry point, Service `proxy-public`. Routes each request to the hub or to the server of the user. |
| `jupyter-<user>` | Pod, one per user | The JupyterLab server of the user, started at sign-in, with the home directory on the volume `claim-<user>`. |

Nothing else: no ingress, no certificate, no identity provider, no database server, no object store.

## Requirements

- A Kubernetes cluster with a default StorageClass.
- About 6 GB free on the node for the first start, while the notebook image is pulled and extracted. The three images then keep about 5 GB.
- The three images below, reachable from the cluster or mirrored in a registry it can pull from.

### Toolchain tested

| Tool | Version |
|---|---|
| Kubernetes (Kind) | 1.30 |
| Kind | 0.23 |
| Helm CLI | 3.18 |
| kubectl | 1.33 |

## Images

| Image | Runs as | What it does | Size, compressed / on disk |
|---|---|---|---|
| `quay.io/jupyterhub/k8s-hub:4.3.3` | the `hub` pod | JupyterHub 5.4 and KubeSpawner, which create the pod and the volume of each user. | 160 MB / 0.6 GB |
| `quay.io/jupyterhub/configurable-http-proxy:5.2.0` | the `proxy` pod | The HTTP proxy in front of everything. The hub adds a route for each user server that starts. | 65 MB / 0.2 GB |
| `quay.io/okdp/jupyter/scipy-notebook:python-3.12.12-hub-5.4.2-lab-4.5.0` | each `jupyter-<user>` pod | JupyterLab 4.5 on Python 3.12 with the scientific stack (numpy, pandas, scipy, scikit-learn, matplotlib), LaTeX for PDF export, and the OKDP additions: `s3fs`, `jupyter-fs`, `jupysql`, `trino`, `nbgitpuller`. Built by [OKDP/jupyterlab-docker](https://github.com/OKDP/jupyterlab-docker). | 1.3 GB / 4.0 GB |

The same list is in [`images.txt`](images.txt), one image per line, for mirroring.

## Installation

```sh
helm repo add jupyterhub https://hub.jupyter.org/helm-chart/
helm install jupyterhub jupyterhub/jupyterhub --version 4.3.3 \
  -f modules/apps/jupyterhub/values/minimal.yaml \
  -n jupyterhub --create-namespace --wait
```

Without Internet access, download the chart archive `https://hub.jupyter.org/helm-chart/jupyterhub-4.3.3.tgz` beforehand and install it in place of `jupyterhub/jupyterhub`.

## Access

```sh
kubectl port-forward svc/proxy-public -n jupyterhub 8080:80
```

Open http://localhost:8080 and sign in with any user name and the password `jupyterhub`. The first server of the cluster takes a few minutes to start while the notebook image is pulled, the next ones take seconds.

## Configuration

Everything is in [`values/minimal.yaml`](values/minimal.yaml). The choices that matter:

| Setting | Value | Why |
|---|---|---|
| `hub.config.JupyterHub.authenticator_class` | `shared-password` | One password for every user name, no identity provider needed. Test installations only. |
| `hub.config.SharedPasswordAuthenticator.user_password` | `jupyterhub` | JupyterHub requires eight characters at least. |
| `hub.config.Authenticator.allow_all` | `true` | JupyterHub 5 refuses every user unless an allow list or this flag is set. |
| `hub.db.type` | `sqlite-pvc` | The hub state stays on a 1 GB volume, no database server. |
| `proxy.service.type` | `ClusterIP` | Reached through `port-forward`, no load balancer needed. |
| `singleuser.uid`, `singleuser.extraEnv` | `0`, `NB_USER`, `CHOWN_HOME` | The notebook image starts as root, creates the user, hands the home directory over to it, then drops to that user. |
| `singleuser.storage` | 1 GB per user, `/home/{username}` | The files of each user survive a server restart. |
| `singleuser.cpu`, `singleuser.memory` | 0.5 to 1 CPU, 1 to 2 GB | Room for a notebook session on a single node cluster. |
| `prePuller`, `cull`, `scheduling` | disabled | Not needed to run, fewer pods and fewer permissions. |

## Verify

```sh
kubectl get pods -n jupyterhub                 # hub and proxy Running
# after a sign-in as "alice" and "Start My Server":
kubectl get pod jupyter-alice -n jupyterhub -o jsonpath='{.status.phase} {.spec.containers[0].image}{"\n"}'
kubectl get pvc -n jupyterhub                  # claim-alice and hub-db-dir Bound
kubectl exec jupyter-alice -n jupyterhub -- ls -ld /home/alice    # owned by alice
```

## Troubleshooting

| Message | Cause | Fix |
|---|---|---|
| Browser: connection refused on `localhost:8080` | The `port-forward` stopped. | Run it again and keep its terminal open. |
| Service Unavailable | The proxy answers, the hub is still starting. | Wait until `hub` is `1/1 Running`, then reload. |
| Invalid username or password | The password is not `jupyterhub`. | Type it again, the user name is free. |
| Login form invalid or expired | The hub restarted since the login page was loaded. | Reload the login page, or open a private window. |
| Spawn failed, `no space left on device` | Not enough disk on the node to extract the notebook image. | Free about 6 GB where the container runtime stores its images. |

## Uninstall

```sh
helm uninstall jupyterhub -n jupyterhub
kubectl delete namespace jupyterhub            # also deletes the volumes of the users
```

## Roadmap

This module covers a standalone JupyterHub. Each integration below will come as a separate values file in `values/`, added to the install command with one more `-f`. `minimal.yaml` stays unchanged.

| Integration | Replaces | Brings |
|---|---|---|
| Identity provider (OIDC) | the shared password | Sign-in with the platform accounts, roles from the platform groups. |
| Ingress and TLS | `port-forward` | A host name served over HTTPS. |
| Database server | sqlite on a volume | The hub state on PostgreSQL. |
| Object storage (S3) | the user volume alone | Shared datasets and notebooks on S3. |
| Spark | Python kernels alone | PySpark kernels and the OKDP Spark notebook images. |

## References

- Chart: [Zero to JupyterHub with Kubernetes](https://z2jh.jupyter.org), chart `jupyterhub` [4.3.3](https://github.com/jupyterhub/zero-to-jupyterhub-k8s/tree/4.3.3).
- Notebook images: [OKDP/jupyterlab-docker](https://github.com/OKDP/jupyterlab-docker).
- Versions: the OKDP package [platform-packages/packages/services/jupyterhub](https://github.com/OKDP/platform-packages/tree/main/packages/services/jupyterhub) pins the same chart and images.
