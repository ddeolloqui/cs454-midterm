# Evidence Log

## 1. Local Application Test

The automated Node.js test was run with:

```bash
npm test
```

The test completed successfully with 1 passing test and 0 failures.

**Screenshot:** `screenshots/01-npm-test.png`

## 2. Docker Compose in Linux VM

Docker Compose was run inside the Kali Linux virtual machine using VMware Workstation.

The project was accessed through the VMware shared folder:

```text
/mnt/hgfs/cs454-midterm
```

The service was started with:

```bash
docker-compose up --build
```

The application started successfully on port 8080, and requests to `/health` and `/` returned successful responses. The application startup and request information were also shown in the container logs.

**Screenshot:** `screenshots/02-compose-kali.png`

## 3. Host-to-VM Access

The Kali Linux VM used IP address `192.168.74.128`.

The Windows host accessed the application running inside the VM with:

```powershell
curl.exe http://192.168.74.128:8080/health
curl.exe http://192.168.74.128:8080/
```

Both requests returned successful responses.

No VMware port forwarding was required because the Windows host could directly reach the VM.

**Screenshot:** `screenshots/03-windows-host-access.png`

## 4. Docker Compose Status

The running Compose container was checked with:

```bash
docker ps
```

The container was running the `cs454-midterm:1.0` image and publishing port 8080.

**Screenshot:** `screenshots/09-compose-status.png`

## 5. Compose Request Count After Restart

The application was accessed before and after restarting the Compose service. The request count was `2` before the restart and returned to `1` after the restart.

This shows that the request count is local to the application process and resets when the process restarts.

**Screenshot:** `screenshots/10-compose-restart.png`

## 6. Kubernetes Deployment

The Kubernetes Deployment was configured with two replicas. Both Pods reached the `Running` state, and the Deployment showed 2/2 available replicas.

The Kubernetes Service was also created on port 8080.

**Screenshot:** `screenshots/04-kubernetes-running.png`

## 7. Kubernetes Service Test

The Kubernetes Service was accessed using port forwarding:

```bash
kubectl port-forward service/cs454-midterm 8081:8080
```

Requests to both `/health` and `/` returned successful responses.

**Screenshot:** `screenshots/05-kubernetes-service-test.png`

## 8. Pod Replacement

One running application Pod was deleted:

```bash
kubectl delete pod cs454-midterm-866fd6947d-74z7g
```

Kubernetes automatically created a replacement Pod because the Deployment was configured to maintain two replicas.

**Screenshot:** `screenshots/06-pod-replacement.png`

## 9. Controlled Service Selector Failure

The working Service selector was:

```yaml
selector:
  app: cs454-midterm
```

To create the controlled failure, the selector was intentionally changed to:

```yaml
selector:
  app: wrong-selector
```

The change was applied with:

```bash
kubectl apply -f k8s/service.yaml
```

The application Pods remained healthy and running, and their labels still included `app=cs454-midterm`. However, the EndpointSlice showed no application endpoints because the Service selector no longer matched the Pod labels.

**Screenshot:** `screenshots/07-broken-selector.png`

## 10. Service Selector Recovery

The Service selector was restored to:

```yaml
selector:
  app: cs454-midterm
```

After applying the corrected configuration, the EndpointSlice showed both application Pod IP addresses:

```text
10.244.0.5,10.244.0.7
```

Requests through the Kubernetes Service were successful again.

This showed that healthy Pods alone do not prove that a client can reach the application through the Service. The Service must also select the correct Pods.

**Screenshot:** `screenshots/08-restored-selector.png`

## Summary

The evidence documents successful local testing, Docker Compose deployment inside the Linux VM, container status and logs, host-to-VM access, process-local request count behavior after a restart, Kubernetes deployment with two ready replicas, Pod replacement, controlled Service selector failure, and recovery.