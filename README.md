A minimal static site served by nginx, built into a Docker image by Jenkins, and deployed to Kubernetes by Argo CD via GitOps.

Repo Structure
pgsql
Copy
Edit
.
├── Dockerfile
├── Jenkinsfile
└── index.html
Tech
nginx (alpine)

Docker

Jenkins (CI)

Argo CD (CD, via config repo)

Kubernetes (Minikube/Kind)

Prerequisites
Docker available on the Jenkins agent

Docker Hub account: rani19

Git provider for this repo and the config repo

Kubernetes cluster (Minikube/Kind)

Argo CD installed on the cluster

Configure Jenkins
Create a Pipeline job pointing to this repo.

Add credentials:

dockerhub-creds: Docker Hub username/password.

git-token: Git Personal Access Token (repo scope) for pushing to the config repo.

Edit Jenkinsfile environment block and set:

DOCKERHUB_USER → rani19

APP_REPO_HTTP → HTTPS URL of this repo

CONFIG_REPO_HTTP → HTTPS URL of the config repo (hello-web-config)

Build Locally (optional)
bash
Copy
Edit
docker build -t rani19/hello-web:dev .
docker run --rm -p 8080:80 rani19/hello-web:dev
# open http://localhost:8080
CI → CD Flow
Push changes to index.html.

Jenkins:

Builds image: rani19/hello-web:<short_sha> and :latest

Pushes to Docker Hub

Clones the config repo and bumps the image tag in k8s/deployment.yaml

Commits & pushes the change

Argo CD detects the change and deploys automatically to the cluster.

Demo (what to show)
Jenkins build log: image built & pushed, config repo updated.

Argo CD UI: hello-web app is Synced/Healthy.

Browser: open http://hello.local (via Ingress) and refresh after each change.

Troubleshooting
Argo app not syncing: check argocd-server access and application targetRevision/path.

Ingress 404/timeout: enable Minikube ingress addon and add 127.0.0.1 hello.local to /etc/hosts.

Image not updating: confirm Jenkins modified k8s/deployment.yaml in the config repo and Argo CD shows a new commit.

ImagePullBackOff: check rani19/hello-web:<tag> exists and the cluster can pull it.

