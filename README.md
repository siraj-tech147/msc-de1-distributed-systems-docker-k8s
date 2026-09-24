# Flask Sample App — Docker & Kubernetes Distributed Systems

## 1. Project Objective

This project takes the original Flask starter application and demonstrates how to containerize, secure, publish, and deploy it as a distributed application using Docker and Kubernetes.

The project demonstrates:

* Flask REST API development
* Docker image creation and security hardening
* Docker Compose
* Docker Hub publication
* Kubernetes deployment using Kind
* Kubernetes Service discovery
* Multiple replicas
* Self-healing
* Horizontal scaling
* Rolling updates
* Rollback
* Container and Kubernetes security
* Vulnerability scanning
* Software Bill of Materials (SBOM)

## 2. Architecture Overview

The application is a Flask REST API running on port `5000`.

```text
                    Docker Hub
                        |
                        v
              Flask Docker Image
              1.0.0 / latest
                        |
                        v
              +-------------------+
              |   Kind Cluster    |
              |                   |
              | Control Plane     |
              |        |          |
              |  +-----+------+   |
              |  |            |   |
              | Worker 1   Worker 2
              |  |            |   |
              | Flask Pods  Flask Pods
              |  +-----+------+   |
              |        |          |
              |   Kubernetes      |
              |     Service       |
              +--------+----------+
                       |
                       v
                  Flask API
                   Port 5000
```

The final Kubernetes deployment uses three Flask replicas distributed across the two worker nodes.

## 3. Original Starter Application

This project was based on the original public Flask starter application:

https://github.com/ubc/flask-sample-app

The starter repository contains the original Flask application, tests, `requirements.txt`, `run.py`, and the original README.

## 4. Project Structure

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

## 5. Prerequisites

Install the following tools:

* Python
* Docker Desktop
* Docker Compose
* Git
* kubectl
* Kind

The Docker Desktop engine must be running for Docker and Kind operations.

## 6. Run the Original Application Locally

Create a virtual environment:

```bash
python -m venv venv
```

On Windows, activate it:

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

### Test the application

Open:

```text
http://localhost:5000/
```

Expected response:

```text
Hello, Flask!
```

Test the items endpoint:

```text
http://localhost:5000/items
```

Expected initial response:

```json
{"items":[]}
```

### Run the tests

```bash
python -m unittest discover tests
```

## 7. Application Routes

| Method | Endpoint           | Description             |
| ------ | ------------------ | ----------------------- |
| GET    | `/`                | Returns `Hello, Flask!` |
| GET    | `/items`           | Returns all items       |
| GET    | `/items/<item_id>` | Returns a specific item |
| POST   | `/items`           | Adds a new item         |

## 8. Build and Run the Docker Image

Build the image:

```bash
docker build -t flask-sample-app:1.0.0 .
```

Run the container:

```bash
docker run -d \
  --name flask-sample-app \
  -p 5000:5000 \
  flask-sample-app:1.0.0
```

Check the container:

```bash
docker ps
```

Check logs:

```bash
docker logs flask-sample-app
```

Test the application:

```text
http://localhost:5000/
```

Expected:

```text
Hello, Flask!
```

Check the health status:

```bash
docker inspect --format "{{.State.Health.Status}}" flask-sample-app
```

Expected:

```text
healthy
```

Verify the container runs as a non-root user:

```bash
docker exec flask-sample-app whoami
```

Expected:

```text
appuser
```

### Docker image inspection

```bash
docker images flask-sample-app
docker history flask-sample-app:1.0.0
docker inspect --format "{{json .Config.ExposedPorts}}" flask-sample-app:1.0.0
```

The image was approximately:

```text
Disk usage: 185 MB
Content size: 44.9 MB
```

## 9. Docker Compose

Start the application:

```bash
docker compose up -d --build
```

Check the service:

```bash
docker compose ps
```

The Compose configuration includes:

* Port mapping `5000:5000`
* Restart policy
* Healthcheck
* `no-new-privileges`
* Dropped Linux capabilities
* Read-only root filesystem
* `/tmp` tmpfs
* Non-secret environment variables

Test the application:

```text
http://localhost:5000/
```

Stop the application:

```bash
docker compose down
```

## 10. Docker Security

The Docker image and Compose configuration use several security controls.

### Non-root execution

The image creates and uses:

```text
appuser
```

Verify:

```bash
docker run --rm flask-sample-app:1.0.0 whoami
```

### Other security controls

* Python slim base image
* Dedicated non-root user
* `no-new-privileges:true`
* All Linux capabilities dropped in Compose
* Read-only root filesystem
* Temporary `/tmp` filesystem
* No privileged mode
* No Docker socket mounting
* No host networking
* `.dockerignore` removes development files and artifacts from the runtime image

## 11. Vulnerability Scan and SBOM

The image was scanned using Trivy.

The scan is stored at:

```text
security/vulnerability-scan.txt
```

The CycloneDX SBOM is stored at:

```text
security/sbom.cdx.json
```

### Recorded scan results

| Component       | Total | Unknown | Low | Medium | High | Critical |
| --------------- | ----: | ------: | --: | -----: | ---: | -------: |
| Base/OS         |   152 |       2 |  57 |     49 |   44 |        0 |
| Python packages |     6 |       0 |   1 |      5 |    0 |        0 |

The image therefore had no critical base/OS findings and no high or critical Python package findings in the recorded scan. The remaining findings are documented in the scan output.

A test using `python:3.12-slim-bookworm` produced a higher number of findings, including critical findings, so that base-image variant was not retained.

## 12. Docker Hub

The public Docker Hub repository is:

https://hub.docker.com/r/sirajuddin147/flask-sample-app

Repository:

```text
sirajuddin147/flask-sample-app
```

Published tags:

```text
1.0.0
latest
```

Pull the versioned image:

```bash
docker pull sirajuddin147/flask-sample-app:1.0.0
```

Run it:

```bash
docker run -d \
  --name flask-hub-test \
  -p 5001:5000 \
  sirajuddin147/flask-sample-app:1.0.0
```

Test:

```text
http://localhost:5001/
```

Clean up:

```bash
docker stop flask-hub-test
docker rm flask-hub-test
```

## 13. Create the Kind Cluster

The Kind configuration creates:

* 1 control-plane node
* 2 worker nodes

Create the cluster:

```bash
kind create cluster --config kind/kind-config.yaml
```

Verify:

```bash
kubectl get nodes -o wide
```

Expected architecture:

```text
flask-cluster-control-plane
flask-cluster-worker
flask-cluster-worker2
```

## 14. Deploy Kubernetes Manifests

Create the namespace:

```bash
kubectl apply -f k8s/namespace.yaml
```

Deploy the application:

```bash
kubectl apply -f k8s/deployment.yaml
```

Create the Service:

```bash
kubectl apply -f k8s/service.yaml
```

Apply the NetworkPolicy:

```bash
kubectl apply -f k8s/network-policy.yaml
```

Check the Deployment:

```bash
kubectl get deployment -n flask-app
```

Check the pods:

```bash
kubectl get pods -n flask-app -o wide
```

Check the Service:

```bash
kubectl get service -n flask-app
```

Check the NetworkPolicy:

```bash
kubectl get networkpolicy -n flask-app
```

## 15. Kubernetes Security

The Kubernetes Deployment uses:

```yaml
runAsNonRoot: true
runAsUser: 1000
runAsGroup: 1000
seccompProfile:
  type: RuntimeDefault
```

The container also uses:

```yaml
allowPrivilegeEscalation: false
capabilities:
  drop:
    - ALL
readOnlyRootFilesystem: true
```

Resource requests and limits are configured:

```text
CPU request:    100m
CPU limit:      500m
Memory request: 128Mi
Memory limit:   256Mi
```

Readiness and liveness probes check:

```text
/
port 5000
```

Verify the security configuration:

```bash
kubectl get deployment flask-app -n flask-app -o yaml
```

Verify the application user:

```bash
kubectl exec -n flask-app deploy/flask-app -- whoami
```

Expected:

```text
appuser
```

## 16. Kubernetes Service and Application Access

The application uses a ClusterIP Service:

```text
flask-service
```

Port:

```text
5000
```

Check endpoints:

```bash
kubectl get endpoints -n flask-app
```

Port-forward the Service:

```bash
kubectl port-forward service/flask-service 8080:5000 -n flask-app
```

Access:

```text
http://localhost:8080/
```

Expected:

```text
Hello, Flask!
```

## 17. NetworkPolicy

The project includes:

```text
k8s/network-policy.yaml
```

Apply:

```bash
kubectl apply -f k8s/network-policy.yaml
```

Verify:

```bash
kubectl get networkpolicy -n flask-app
```

The policy allows ingress to TCP port `5000`.

### Known limitation

NetworkPolicy enforcement depends on the Kubernetes networking/CNI implementation.

The fact that Kubernetes accepts the NetworkPolicy object does not by itself prove that traffic filtering is enforced.

The local Kind environment should therefore be treated as having the policy configured, while enforcement should be tested with an enforcing CNI before using the configuration as a production security control.

## 18. Distributed Behavior

### Multiple replicas

The final Deployment uses three replicas:

```bash
kubectl get pods -n flask-app -o wide
```

The final pods were distributed across the two worker nodes.

### Self-healing

Delete a running pod:

```bash
kubectl delete pod <pod-name> -n flask-app
```

Then check:

```bash
kubectl get pods -n flask-app -o wide
```

Kubernetes automatically creates a replacement pod to restore the desired replica count.

### Scaling

Scale from two to three replicas:

```bash
kubectl scale deployment flask-app --replicas=3 -n flask-app
```

Verify:

```bash
kubectl get deployment flask-app -n flask-app
kubectl get pods -n flask-app -o wide
```

### Rolling update

The rolling update demonstration used the `latest` image:

```bash
kubectl set image deployment/flask-app \
flask-app=sirajuddin147/flask-sample-app:latest \
-n flask-app
```

Monitor:

```bash
kubectl rollout status deployment/flask-app -n flask-app
```

View history:

```bash
kubectl rollout history deployment/flask-app -n flask-app
```

### Rollback

Rollback:

```bash
kubectl rollout undo deployment/flask-app -n flask-app
```

Verify:

```bash
kubectl rollout status deployment/flask-app -n flask-app
```

The final deployment was explicitly restored to the versioned image:

```text
sirajuddin147/flask-sample-app:1.0.0
```

Verify:

```bash
kubectl get deployment flask-app -n flask-app \
-o jsonpath="{.spec.template.spec.containers[0].image}"
```

## 19. Evidence

Supporting evidence is stored in:

```text
evidence/screenshots-or-command-output/
```

The evidence includes:

* Kind node information
* Deployment configuration
* Deployment security configuration
* Pod information
* Service configuration
* NetworkPolicy
* Rollout history
* Final image

Security evidence is stored in:

```text
security/
```

## 20. Cleanup

Delete the application namespace:

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

## 21. Git Repository

GitHub repository:

https://github.com/siraj-tech147/msc-de1-distributed-systems-docker-k8s

Check repository status:

```bash
git status
```

The final working tree should be clean before submission.

## 22. Production Improvement

For a production deployment, the in-memory item list should be replaced with a persistent external database.

This would prevent application data from being lost when Kubernetes recreates a pod. Production deployment should also add centralized logging, metrics, monitoring, and an enforcing Kubernetes networking solution.

## 23. License

This project is provided for educational purposes under the included MIT License.
