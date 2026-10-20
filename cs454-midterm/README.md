# CS 454 Midterm Mini-Project

## Overview

This project deploys the same dependency-free Node.js HTTP service in two environments:

1. Docker Compose inside a Kali Linux virtual machine
2. Local Kubernetes

The service provides:

- `GET /` for the service name, message, hostname, and process-local request count
- `GET /health` for a health check returning HTTP 200

The application uses port `8080`.

## Project Structure

```text
cs454-midterm/
├── src/
│   └── server.js
├── test/
│   ├── service.test.js
│   └── smoke.sh
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── screenshots/
├── .dockerignore
├── .gitignore
├── .nvmrc
├── Dockerfile
├── compose.yaml
├── package.json
├── evidence.md
├── report.md
└── README.md
```

## Prerequisites

- Node.js 22+
- Docker
- Docker Compose
- `kubectl`
- VMware Workstation with Kali Linux for the Compose deployment
- Docker Desktop with local Kubernetes enabled

## Local Testing

```bash
npm test
```

The automated tests passed successfully.

The application can also be started locally with:

```bash
npm start
```

## Docker Compose

The project was run inside a Kali Linux VM using VMware Workstation. The project was accessed through a VMware shared folder at:

```text
/mnt/hgfs/cs454-midterm
```

Start the application:

```bash
docker-compose up --build
```

Test from inside the VM:

```bash
curl http://localhost:8080/health
curl http://localhost:8080/
```

The Kali VM used IP address `192.168.74.128`. The Windows host could access the application with:

```powershell
curl.exe http://192.168.74.128:8080/health
curl.exe http://192.168.74.128:8080/
```

No VMware port forwarding was required.

Check the running container:

```bash
docker ps
```

The request count was also tested before and after `docker-compose restart`. The count reset because it is local to the application process.

## Kubernetes

The Deployment uses two replicas and the same `cs454-midterm:1.0` image used by Compose.

The local Kubernetes cluster is provided by Docker Desktop. The image was built in Docker Desktop's local Docker environment, so the cluster could use the local image with `imagePullPolicy: IfNotPresent`. No public registry was required.

Deploy:

```bash
kubectl apply -f k8s/
```

Check the Deployment, Pods, and Service:

```bash
kubectl get deployment,pods,service
```

Access the Service locally:

```bash
kubectl port-forward service/cs454-midterm 8081:8080
```

Then, from another terminal:

```powershell
curl.exe http://localhost:8081/health
curl.exe http://localhost:8081/
```

One Pod was deleted during testing, and Kubernetes automatically created a replacement to maintain two replicas.

## Controlled Troubleshooting

The working Service selector was:

```yaml
selector:
  app: cs454-midterm
```

For the controlled failure, the selector was changed to:

```yaml
selector:
  app: wrong-selector
```

The Pods remained healthy and still had the `app=cs454-midterm` label, but the EndpointSlice had no application endpoints because the Service selector no longer matched the Pods.

The selector was then restored and the EndpointSlice returned the application Pod IP addresses. Service requests worked again.

## Evidence and Report

`evidence.md` contains the labeled screenshots and terminal evidence for the required tests.

`report.md` contains the deployment comparison, troubleshooting analysis, and OpenStack deployment plan.

## Video Walkthrough

[CS 454 Midterm Video Walkthrough](https://drive.google.com/file/d/1X1DF9-4VIKfSDblytz8IZWY9NG3ny9uz/view?usp=sharing)

## Cleanup

```bash
kubectl delete -f k8s/
docker-compose down
```