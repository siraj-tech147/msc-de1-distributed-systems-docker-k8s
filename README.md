# Flask Sample App — Docker & Kubernetes Distributed Systems

A containerized Flask REST API demonstrating Docker, Docker Compose, Docker Hub, Kubernetes, Kind, security hardening, service discovery, scaling, self-healing, rolling updates, and rollback.

## Project Overview

This project uses a simple Flask application that provides a REST API for managing a list of items.

The application exposes:

* `GET /` — Returns a greeting message.
* `GET /items` — Returns all items.
* `GET /items/<item_id>` — Returns a specific item.
* `POST /items` — Adds an item.

The application runs on port `5000`.

## Project Structure

```text
flask-sample-app/
│
├── app/
│   ├── __init__.py
│   └── routes.py
│
├── tests/
│   ├── __init__.py
│   └── test_app.py
│
├── evidence/
│   └── screenshots-or-command-output/
│       ├── deployment-security.yaml
│       ├── deployment.txt
│       ├── final-image.txt
│       ├── kind-nodes.txt
│       ├── network-policy.txt
│       ├── pods.txt
│       ├── rollout-history.txt
│       └── service.txt
│
├── k8s/
│   ├── deployment.yaml
│   ├── namespace.yaml
│   ├── network-policy.yaml
│   └── service.yaml
│
├── kind/
│   └── kind-config.yaml
│
├── security/
│   ├── sbom.cdx.json
│   └── vulnerability-scan.txt
│
├── .dockerignore
├── .gitignore
├── compose.yaml
├── Dockerfile
├── LICENSE
├── README.md
├── requirements.txt
└── run.py
```

## Requirements

The project requires:

* Python
* Docker Desktop
* Docker Compose
* kubectl
* Kind
* Git

## Run Locally

Create a virtual environment:

```bash
python -m venv venv
```

Activate it on Windows:

```powershell
.\venv\Scripts\Activate.ps1
```

Install dependencies:

```bash
pip install -r requirements.txt
```

Run the Flask application:

```bash
python run.py
```

The application runs on:

```text
http://localhost:5000
```

## Run Tests

Run the application tests with:

```bash
python -m unittest discover tests
```

## Docker

### Build the Image

```bash
docker build -t flask-sample-app:1.0.0 .
```

### Run the Container

```bash
docker run -d --name flask-sample-app -p 5000:5000 flask-sample-app:1.0.0
```

Verify the application:

```bash
curl http://localhost:5000/
```

Expected response:

```text
Hello, Flask!
```

Check the container:

```bash
docker ps
docker logs flask-sample-app
```

Check the health status:

```bash
docker inspect --format "{{.State.Health.Status}}" flask-sample-app
```

The container is configured with a Docker healthcheck.

### Docker Security

The Docker image uses:

* Python slim base image
* A dedicated non-root `appuser`
* `USER appuser`
* `no-new-privileges`
* All Linux capabilities dropped in Compose
* Read-only root filesystem in Compose
* Temporary filesystem for `/tmp`
* Minimal application files copied into the image
* `.dockerignore` to exclude development files and artifacts

Verify the runtime user:

```bash
docker run --rm flask-sample-app:1.0.0 whoami
```

Expected:

```text
appuser
```

## Docker Compose

Start the application with:

```bash
docker compose up -d --build
```

Check the service:

```bash
docker compose ps
```

The Compose configuration includes:

* Application build configuration
* Port mapping `5000:5000`
* Restart policy
* Environment variables
* Healthcheck
* `no-new-privileges`
* Dropped capabilities
* Read-only root filesystem
* `/tmp` tmpfs

Stop the application:

```bash
docker compose down
```

## Image Inspection

Inspect the image history:

```bash
docker history flask-sample-app:1.0.0
```

Inspect exposed ports:

```bash
docker inspect --format "{{json .Config.ExposedPorts}}" flask-sample-app:1.0.0
```

The application exposes only:

```text
5000/tcp
```

## Vulnerability Scanning and SBOM

The container image was scanned using Trivy.

The scan results are stored in:

```text
security/vulnerability-scan.txt
```

The generated CycloneDX SBOM is stored in:

```text
security/sbom.cdx.json
```

The scan should be reviewed together with the recorded severity counts in the security evidence. The presence of reported vulnerabilities does not mean the image is vulnerability-free; the scan results document the identified findings and their severity.

## Docker Hub

The published Docker image is:

```text
siraj-tech147/flask-sample-app
```

Available tags include:

```text
1.0.0
latest
```

The Kubernetes deployment uses the documented versioned image:

```text
siraj-tech147/flask-sample-app:1.0.0
```

Pull the image with:

```bash
docker pull siraj-tech147/flask-sample-app:1.0.0
```

Run the Docker Hub image:

```bash
docker run -d --name flask-hub-test -p 5001:5000 siraj-tech147/flask-sample-app:1.0.0
```

Verify:

```bash
curl http://localhost:5001/
```

Clean up:

```bash
docker stop flask-hub-test
docker rm flask-hub-test
```

## Kubernetes with Kind

The project uses a three-node Kind cluster:

* 1 control-plane node
* 2 worker nodes

Create the cluster:

```bash
kind create cluster --config kind/kind-config.yaml
```

Verify the nodes:

```bash
kubectl get nodes -o wide
```

## Kubernetes Namespace

Create the application namespace:

```bash
kubectl apply -f k8s/namespace.yaml
```

Verify:

```bash
kubectl get namespace flask-app
```

## Kubernetes Deployment

Apply the Deployment:

```bash
kubectl apply -f k8s/deployment.yaml
```

The Deployment provides:

* 3 application replicas in the final state
* RollingUpdate strategy
* CPU and memory requests
* CPU and memory limits
* Readiness probe
* Liveness probe
* Non-root execution
* `runAsNonRoot`
* Explicit UID/GID
* `allowPrivilegeEscalation: false`
* All capabilities dropped
* `seccompProfile: RuntimeDefault`
* Read-only root filesystem
* Temporary `/tmp` volume

Check the Deployment:

```bash
kubectl get deployment flask-app -n flask-app
```

Check the pods:

```bash
kubectl get pods -n flask-app -o wide
```

## Kubernetes Service

Apply the Service:

```bash
kubectl apply -f k8s/service.yaml
```

Check the Service:

```bash
kubectl get service -n flask-app
```

The Service is a `ClusterIP` service exposing port `5000`.

Check service endpoints:

```bash
kubectl get endpoints -n flask-app
```

The Service provides stable internal access to the Flask application while Kubernetes distributes traffic across available pods.

### Port Forwarding

Forward the Kubernetes Service to the local machine:

```bash
kubectl port-forward service/flask-service 8080:5000 -n flask-app
```

The application can then be accessed at:

```text
http://localhost:8080/
```

## Kubernetes NetworkPolicy

The project includes:

```text
k8s/network-policy.yaml
```

Apply it with:

```bash
kubectl apply -f k8s/network-policy.yaml
```

Verify:

```bash
kubectl get networkpolicy -n flask-app
```

The policy defines ingress rules for TCP port `5000`.

NetworkPolicy enforcement depends on the networking implementation used by the cluster. The policy object being accepted by the Kubernetes API does not by itself prove that the local Kind networking environment enforces every NetworkPolicy rule. The evidence therefore records the policy configuration separately.

## Distributed Systems Demonstrations

### Multiple Replicas

The Deployment runs multiple Flask replicas:

```bash
kubectl get pods -n flask-app -o wide
```

The pods can be scheduled across the two worker nodes.

### Self-Healing

A running pod was deleted manually:

```bash
kubectl delete pod <pod-name> -n flask-app
```

Kubernetes automatically created a replacement pod to maintain the desired replica count.

### Scaling

The application was scaled from 2 replicas to 3 replicas:

```bash
kubectl scale deployment flask-app --replicas=3 -n flask-app
```

Verify:

```bash
kubectl get deployment flask-app -n flask-app
kubectl get pods -n flask-app -o wide
```

### Rolling Update

A rolling update was demonstrated by updating the application image:

```bash
kubectl set image deployment/flask-app flask-app=sirajuddin147/flask-sample-app:latest -n flask-app
```

Monitor the rollout:

```bash
kubectl rollout status deployment/flask-app -n flask-app
```

View rollout history:

```bash
kubectl rollout history deployment/flask-app -n flask-app
```

### Rollback

A rollback was also demonstrated:

```bash
kubectl rollout undo deployment/flask-app -n flask-app
```

After the demonstration, the final deployment was restored to the versioned image:

```text
sirajuddin147/flask-sample-app:1.0.0
```

Verify the final image:

```bash
kubectl get deployment flask-app -n flask-app -o jsonpath="{.spec.template.spec.containers[0].image}"
```

Expected:

```text
sirajuddin147/flask-sample-app:1.0.0
```

## Evidence

Command outputs and deployment evidence are stored in:

```text
evidence/screenshots-or-command-output/
```

Important evidence includes:

* Kind cluster nodes
* Kubernetes deployment configuration
* Kubernetes security configuration
* Pods and node placement
* Service configuration
* NetworkPolicy
* Rollout history
* Final Kubernetes image

## Cleanup

Remove the Kubernetes application:

```bash
kubectl delete namespace flask-app
```

Delete the Kind cluster:

```bash
kind delete cluster --name flask-cluster
```

Stop Docker Compose:

```bash
docker compose down
```

Remove unused local containers if required:

```bash
docker container prune
```

## Git

The project is maintained using Git with separate commits for the initial application and the Docker/Kubernetes work.

Check repository status:

```bash
git status
```

The final working tree should be clean before submission.

## Author

Sirajuddin Shaik

## License

This project is provided for educational purposes under the included MIT License.
