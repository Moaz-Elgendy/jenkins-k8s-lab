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

### Step 1 — Join the agent to the minikube network

The agent and minikube are on separate docker networks, so the agent cannot route to the apiserver at all. No container recreate needed:

```bash
docker network connect minikube strange_babbage
docker exec -u root strange_babbage apt-get update
docker exec -u root strange_babbage apt-get install -y curl
# verify: should print HTTP 403 (TLS ok, unauthenticated request rejected)
docker exec -u root strange_babbage curl -sk -o /dev/null -w '%{http_code}\n' https://192.168.49.2:8443
```

### Step 2 — Install kubectl in the agent

```bash
docker exec -u root strange_babbage apt-get install -y kubectl
# if the distro repo is too old, copy the host binary instead:
#   docker cp /usr/bin/kubectl strange_babbage:/usr/local/bin/kubectl
docker exec strange_babbage kubectl version --client
```

### Step 3 — Add the kubeconfig as a Jenkins Secret File credential

A ready-to-use kubeconfig already exists at:

```
/home/moaz/.kube/jenkins-kubeconfig   (mode 600, not in git)
```

It embeds the minikube client certificate inline (`certificate-authority-data`,
`client-certificate-data`, `client-key-data`) instead of referencing host file paths, so it is
self-contained inside the container. Its `server:` is already rewritten to
`https://192.168.49.2:8443`.

In Jenkins: **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

| Field | Value |
|---|---|
| Kind | Secret file |
| Scope | Global |
| ID | `minikube-kubeconfig` |
| File | paste the contents of `/home/moaz/.kube/jenkins-kubeconfig` |

> The embedded identity is `CN=minikube-user`, `O=system:masters` — i.e. **cluster-admin**.
> That is what the lab guide asks for (Step 3: import the minikube kubeconfig). Acceptable on a
> local single-user minikube; do not reuse this pattern against a shared or production cluster.
> The file is a private key, so it stays outside the git working tree on purpose — never commit it.

### Step 4 — Set `KUBECONFIG_PATH`

If you used the Secret File credential, the pipeline needs to consume it. The current `Jenkinsfile`
reads a plain path, so either copy the secret into the agent at a fixed path:

```bash
docker cp /home/moaz/.kube/jenkins-kubeconfig strange_babbage:/var/jenkins_home/.kube/config
docker exec strange_babbage mkdir -p /var/jenkins_home/.kube
docker exec -u root strange_babbage chown -R jenkins:jenkins /var/jenkins_home/.kube
docker exec strange_babbage ls -l /var/jenkins_home/.kube/config
```

then leave `KUBECONFIG_PATH` at its default `/home/jenkins/.kube/config`... note that the agent's
`$HOME` is `/var/jenkins_home`, so set the parameter to `/var/jenkins_home/.kube/config`.

Or switch the `Deploy to Kubernetes` and `Verify Deployment` stages to read the credential directly:

```groovy
withCredentials([file(credentialsId: 'minikube-kubeconfig', variable: 'KUBECONFIG')]) {
    sh "kubectl --kubeconfig=\"${KUBECONFIG}\" apply -f rendered-deployment.yaml"
}
```

### Alternative fixes

**`minikube tunnel`** — publishes the apiserver on the host; still needs kubectl in the agent.

**Move the deploy off the agent** — run deploy from the workstation; simplest, but the pipeline is no
longer self-contained.

## Verify manually

```bash
minikube status
kubectl get nodes
kubectl get pods -o wide
kubectl get svc
curl http://$(minikube ip):30080/
```