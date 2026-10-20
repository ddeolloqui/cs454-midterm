# CS 454 Midterm Mini-Project
## One Service, Two Deployment Environments

### Overview

This project deploys the same Node.js HTTP service in two environments: Docker Compose inside a Kali Linux virtual machine and a local Kubernetes cluster running through Docker Desktop. The project demonstrates how the same container image can be used in both environments and how each platform handles service management and recovery.

### Deployment Diagram

```text
Windows Host
    |
    | HTTP :8080
    v
Kali Linux VM
192.168.74.128
    |
    | Docker Compose
    v
Container
cs454-midterm:1.0
    |
    v
Node.js Application
port 8080
```

For Kubernetes, the local cluster runs through Docker Desktop on the Windows host:

```text
Windows Host
    |
    | kubectl port-forward :8081 -> :8080
    v
Kubernetes Service
cs454-midterm
    |
    v
2 Kubernetes Pods
    |
    v
Node.js Application
port 8080
```

### Docker Compose Deployment

Docker Compose was run inside a Kali Linux VM using VMware Workstation. The project was accessed through a VMware shared folder at `/mnt/hgfs/cs454-midterm`.

The Compose configuration builds and runs the `cs454-midterm:1.0` image, sets the application environment variables, and publishes port 8080. The service successfully responded to both `/health` and `/` from inside the VM.

The Windows host was also able to access the service using the VM's IP address, `192.168.74.128`. No VMware port forwarding was required because the host could directly reach the VM over the local network.

The container status and logs were also checked. The logs showed the application starting on port 8080 and recording HTTP requests.

The application keeps its request count in the Node.js process. Before restarting the Compose service, the request count was 2. After `docker-compose restart`, the next request returned a count of 1. This demonstrates that the counter is process-local and is reset when the application process restarts.

### Kubernetes Deployment

The same `cs454-midterm:1.0` image was used for the Kubernetes deployment. The local Kubernetes cluster runs through Docker Desktop on the Windows host. The image was built in Docker Desktop's local Docker environment, so Kubernetes could use the existing local image with `imagePullPolicy: IfNotPresent`. A public image registry was not required.

The Deployment was configured with two replicas, and the Service selects Pods using the `app: cs454-midterm` label. Both Pods reached the Running state and the Deployment showed two available replicas.

The Service was tested with:

```bash
kubectl port-forward service/cs454-midterm 8081:8080
```

Requests to `/health` and `/` returned successful responses. Port forwarding demonstrates that the Service can provide access to the application, but it does not prove that requests are being distributed between both replicas.

One application Pod was then deleted manually. Kubernetes created a replacement Pod because the Deployment's desired state specifies two replicas. Once the replacement became ready, the deployment again had two running replicas. This differs from the Compose restart test because Kubernetes automatically reconciles the desired number of replicas, while Compose does not automatically maintain multiple replicas in this configuration.

### Controlled Troubleshooting

The working Kubernetes Service used this selector:

```yaml
selector:
  app: cs454-midterm
```

For the controlled failure, the selector was intentionally changed to:

```yaml
selector:
  app: wrong-selector
```

The Service was reapplied while the application Pods remained healthy. The Pods still had the `app=cs454-midterm` label, but the EndpointSlice showed no application endpoints because the Service selector no longer matched the Pod labels.

The failure demonstrated that healthy Pods alone do not prove Service connectivity. The Service must correctly select the Pods that should receive traffic.

The selector was then restored to `app: cs454-midterm`. The EndpointSlice showed both application Pod IP addresses again, and requests through the Service succeeded.

### OpenStack Deployment Analysis

For an OpenStack deployment, I would first create or use a small Linux VM to run Docker and Docker Compose. The workload is lightweight, so a modest flavor with a small amount of CPU and memory would be sufficient. I would use an approved Linux image from Glance, create or use a key pair for SSH access, and place the instance on a tenant network and subnet. The tenant network would connect through a router to an external network so the instance could receive a floating IP for external access.

The main resources would be the VM image, flavor, key pair, tenant network, subnet, router, security group, and floating IP. Some of these resources, especially the external network and possibly the tenant network, could already be provided by the cloud administrator. The application does not need persistent storage because its request counter is intentionally process-local and the service does not store application data. Therefore, a Cinder volume is not necessary for this workload.

Several OpenStack services are involved in creating this environment. Keystone provides identity and authentication for OpenStack operations. Glance provides the VM image used to create the instance. Nova is responsible for creating and managing the compute instance. Neutron provides the networking components, including the tenant network, subnet, router, security group, and floating IP connectivity. After the VM is running, Docker Compose is responsible for building and running the application container. Cinder would only be needed if persistent block storage were required.

The request path would be:

```text
External Client
      |
      v
Floating IP
      |
      v
Neutron / Security Group
      |
      v
OpenStack VM
      |
      v
Docker Compose
      |
      v
Container :8080
      |
      v
Node.js Application
```

The security group would need an inbound TCP rule for port 8080 so external HTTP requests could reach the application. SSH would use a separate TCP port 22 rule for administration. ICMP could optionally be allowed for troubleshooting with ping, but it is not required for the application itself.

If the instance were running but an external client could not reach the service, I would troubleshoot from the inside out. First, I would check whether the container is running with `docker ps`, inspect its logs, and test `curl http://localhost:8080/` from inside the VM. If that failed, the problem would likely be at the application or container layer. If the service worked inside the VM, I would check that Docker published port 8080 and that the VM was listening on the expected address and port. I would then check the OpenStack security group, floating IP association, router, subnet, and network connectivity. This separates an application problem from a cloud networking problem and provides evidence at each layer.

Operating an OpenStack cloud is more complex than simply deploying an application into an already working cloud because multiple infrastructure services must work together. A problem with identity, images, compute, networking, or security groups can prevent an otherwise working application from being reachable. This is different from an application deployment problem, such as an incorrect Docker port or a failed container. The course Kubernetes demonstration also showed this distinction: Kubernetes manages application state after the cluster is available, while infrastructure setup is responsible for providing the environment in which the application can run.

### Final Comparison

Docker Compose was sufficient for this small application because there was only one container and no need for automatic replication. Kubernetes provided additional behavior by maintaining two replicas, replacing a deleted Pod automatically, and providing a Service with stable access to the application. Kubernetes therefore provides more built-in orchestration and recovery, while Compose is simpler for a small single-service deployment.

### References

- UAH CS 454/554 Midterm Mini-Project assignment instructions and course materials.
- UAH CS 454/554 course exercise: *From Containers to Orchestration with Local Kubernetes*.