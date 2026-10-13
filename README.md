# jenkins-k8s-lab

Jenkins pipeline that builds a Flask app, pushes the image to Docker Hub, and deploys it to a local **minikube** cluster.

## Repository layout

```
.
├── Dockerfile              # python:3.12-slim, installs requirements.txt, runs hello.py
├── hello.py                # Flask app on :5000 (also exposes /health)
├── requirements.txt        # flask
├── Jenkinsfile             # pipeline: build -> push -> deploy -> verify
├── .dockerignore
└── k8s/
    ├── deployment.yaml     # Deployment, image is IMAGE_PLACEHOLDER at rest
    └── service.yaml        # NodePort 30080 -> 5000
```

> **Deviation from the lab guide:** the PDF calls these `Python.py`, `req.txt`, and `K8s/`. This repo keeps the names that actually work and match the existing branch (`hello.py`, `requirements.txt`, `k8s/`).

## How the image reaches the cluster

The image is **pushed to Docker Hub** and pulled by minikube. `deployment.yaml` therefore sets:

```yaml
imagePullPolicy: Always
```

The pipeline substitutes the tag at build time:

```
IMAGE_PLACEHOLDER  ->  <dockerhub-user>/new-app-nti:<BUILD_NUMBER>
```

Consequence: the Docker Hub repo must be **public**, or the cluster needs an `imagePullSecret`.

## Pipeline stages

| Stage | What it does |
|---|---|
| Getting Repo files | `git` checkout using `SCM_CRED_ID` |
| Build Docker Image | `docker build -t new-app-nti:$BUILD_NUMBER .` |
| Push Docker Image | `docker login` + `tag` + `push` using `DOCKER_CRED_ID` |
| Prepare Manifests | `sed` substitutes `IMAGE_PLACEHOLDER`, fails loudly if it did not apply |
| Deploy to Kubernetes | `kubectl apply` with `KUBECONFIG_PATH` |
| Verify Deployment | `kubectl rollout status`, then prints pods / services / app URL |

## Job parameters

| Parameter | Default | Notes |
|---|---|---|
| `BRANCH` | `main` | |
| `REPO_URL` | this repo | |
| `SCM_CRED_ID` | `jenkins-token` | exists in this Jenkins |
| `DOCKER_CRED_ID` | `docker` | exists in this Jenkins |
| `KUBECONFIG_PATH` | `/home/jenkins/.kube/config` | **must be changed**, see below |

## ⚠️ Agent prerequisites — NOT yet satisfied

The Jenkins agent is a container (`strange_babbage`, `jenkins/jenkins:lts`). Verified gaps:

1. **`kubectl` is not installed in the agent.** Only `docker`, `git`, `terraform` are on PATH.
2. **No kubeconfig is mounted into the agent.**
3. **The agent cannot reach the apiserver even with a kubeconfig.** The agent sits on the docker `bridge` network (`172.17.0.2`); minikube sits on its own `minikube` network (`192.168.49.2`). `curl https://192.168.49.2:8443` from the agent returns HTTP 000.
4. **The apiserver is bound to host loopback only** — `8443/tcp -> 127.0.0.1:32771` — so the host gateway route is closed as well.

### Fix options

**A. Join the agent to the minikube network** (minimal, no container recreate)

```bash
docker network connect minikube strange_babbage
docker exec strange_babbage apt-get update && \
docker exec strange_babbage apt-get install -y curl
# then test:
docker exec strange_babbage curl -sk https://192.168.49.2:8443
```

Then generate a kubeconfig whose `server:` is `https://192.168.49.2:8443` (rewrite `127.0.0.1:32771`) and add it as a Jenkins **Secret File** credential, or copy it into the agent.

**B. Use `minikube tunnel`** — publishes the apiserver on the host, still needs the binary in the agent.

**C. Move the deploy off the agent** — run the deploy from the workstation; simplest, but the pipeline is no longer self-contained.

> Note: the guide's Step 3 says to import the kubeconfig as a Jenkins Secret File credential. That is necessary either way, but on its own it is **not sufficient** here — the network reachability issue (item 3) has to be solved too.

## Verify manually

```bash
minikube status
kubectl get nodes
kubectl get pods -o wide
kubectl get svc
curl http://$(minikube ip):30080/
```