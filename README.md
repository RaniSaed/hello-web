# hello-web

A minimal static site served by **nginx**, built into a Docker image by **Jenkins**, and deployed to **Kubernetes** by **Argo CD** via **GitOps**.

---

## Repo Structure

```
.
├── Dockerfile
├── Jenkinsfile
└── index.html
```

## Tech

* nginx (alpine)
* Docker
* Jenkins (CI)
* Argo CD (CD, via config repo)
* Kubernetes (Minikube/Kind)

## Prerequisites

* Docker available on the Jenkins agent
* Docker Hub account: **rani19**
* Git provider for **this repo** and the **config repo** (e.g. `hello-web-config`)
* A local Kubernetes cluster (Minikube or Kind)
* Argo CD installed on the cluster (app points to the config repo)

---

## Configure Jenkins

1. **Create a Pipeline job** that points to this repo (Multibranch or single-branch Pipeline is fine).
2. **Credentials** (add in Jenkins > Manage Credentials):

   * `dockerhub-creds`: Docker Hub username/password.
   * `git-token`: Git Personal Access Token (repo scope) for pushing to the **config repo**.
3. **Environment variables** (in the `Jenkinsfile` or job env):

   * `DOCKERHUB_USER` → `rani19`
   * `APP_REPO_HTTP` → HTTPS URL of **this** repo
   * `CONFIG_REPO_HTTP` → HTTPS URL of the **config repo** (`hello-web-config`)

> The pipeline:
>
> * Builds and tags `rani19/hello-web:<short_sha>` and `rani19/hello-web:latest`
> * Pushes to Docker Hub
> * Clones the config repo and bumps the image tag in `k8s/deployment.yaml`
> * Commits & pushes the change → Argo CD deploys

---

## Build Locally (optional)

```bash
docker build -t rani19/hello-web:dev .
docker run --rm -p 8080:80 rani19/hello-web:dev
# open http://localhost:8080
```

---

## CI → CD Flow

1. **Edit** `index.html` and **push**.
2. **Jenkins**:

   * Builds image `rani19/hello-web:<short_sha>` and `rani19/hello-web:latest`
   * Pushes to Docker Hub
   * Updates `k8s/deployment.yaml` in the **config repo** with the new image tag
   * Commits & pushes
3. **Argo CD** detects the new commit in the **config repo** and **syncs** the cluster.

---

## Demo (what to show)

* **Jenkins build log**: image built & pushed, config repo updated.
* **Argo CD UI**: `hello-web` app is **Synced/Healthy**.
* **Browser**: open `http://hello.local` (via Ingress) and refresh after each change.

---

## Kubernetes / Ingress Notes (Minikube)

Enable ingress and map a host:

```bash
minikube addons enable ingress
# Add to /etc/hosts (or Windows hosts file):
# 127.0.0.1  hello.local
```

Then open:

```
http://hello.local
```

---

## Troubleshooting

**Argo app not syncing**

* Verify Argo CD can reach the repo (auth), `targetRevision` and `path` in the Application are correct.
* `argocd app get hello-web` to see status.

**Ingress 404/timeout**

* Ensure Minikube ingress addon is enabled.
* Confirm `Service` type/ports and `Ingress` host `hello.local` match.
* Add `127.0.0.1 hello.local` to `/etc/hosts` (or your OS equivalent).

**Image not updating in cluster**

* Check Jenkins really modified `k8s/deployment.yaml` in the **config repo** (new image tag).
* Confirm Argo CD shows the new commit and has synced.
* `kubectl -n <ns> rollout status deploy/hello-web` to watch rollout.

**ImagePullBackOff**

* Verify `rani19/hello-web:<tag>` exists on Docker Hub.
* Check the cluster has internet access (for image pulls).
* If using a private repo, configure an imagePullSecret and reference it in the Deployment.

---

## Quick Commands

```bash
# Check current image in the running Deployment
kubectl -n <ns> get deploy hello-web -o=jsonpath='{.spec.template.spec.containers[0].image}'

# Force a sync from CLI (if needed)
argocd app sync hello-web --prune

# Tail pod logs
kubectl -n <ns> logs -l app=hello-web -f

# Port-forward Service (debug)
kubectl -n <ns> port-forward svc/hello-web 8080:80
```

---

## Repos

* **App repo (this one)**: Dockerfile, Jenkinsfile, static `index.html`
* **Config repo**: Argo CD `Application` + Kubernetes manifests (`k8s/deployment.yaml`, `service.yaml`, `ingress.yaml`, `namespace.yaml`) pointing to `rani19/hello-web:<tag>`
