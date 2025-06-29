# Kubernetes/AKS Operations Guide

---

## Table of Contents

* [Kubernetes/AKS Deployments: Replica Sets, Replication Controllers, and Deployments](#kubernetesaks-deployments-replica-sets-replication-controllers-and-deployments)
    * [Replica Set](#replica-set)
    * [Replication Controller](#replication-controller)
    * [Deployment](#deployment)
    * [Comparison: ReplicaSet vs. Deployment](#comparison-replicaset-vs-deployment)
* [Kubernetes Service Types](#kubernetes-service-types)
    * [ClusterIP Service](#clusterip-service)
    * [NodePort Service](#nodeport-service)
    * [LoadBalancer Service](#loadbalancer-service)
* [PersistentVolume (PV) and PersistentVolumeClaim (PVC)](#persistentvolume-pv-and-persistentvolumeclaim-pvc)
    * [PersistentVolume (PV)](#persistentvolume-pv)
    * [PersistentVolumeClaim (PVC)](#persistentvolumeclaim-pvc)
    * [Attaching PVC to a Pod](#attaching-pvc-to-a-pod)
* [Managing Kubernetes with Azure Kubernetes Service (AKS)](#managing-kubernetes-with-azure-kubernetes-service-aks)
    * [Creating an AKS Cluster](#creating-an-aks-cluster)
    * [Connecting to an AKS Cluster](#connecting-to-an-aks-cluster)
    * [Scaling an AKS Cluster](#scaling-an-aks-cluster)
    * [Upgrading an AKS Cluster](#upgrading-an-aks-cluster)
* [Configure Liveness and Readiness Probes for Pods](#configure-liveness-and-readiness-probes-for-pods)
    * [Liveness Probe](#liveness-probe)
    * [Readiness Probe](#readiness-probe)
* [Configure Taints and Tolerations](#configure-taints-and-tolerations)
    * [Taints](#taints)
    * [Tolerations](#tolerations)
* [Configure Autoscaling in Your Cluster (Horizontal Scaling)](#configure-autoscaling-in-your-cluster-horizontal-scaling)

---

## Kubernetes/AKS Deployments: Replica Sets, Replication Controllers, and Deployments

These controllers ensure that a specified number of identical pods are running.

### Replica Set

A **ReplicaSet** maintains a stable set of replica Pods running at any given time. It ensures that a specified number of identical pods are always running.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
  labels:
    app: myapp
spec:
  replicas: 3 # Desired number of pods
  selector:
    matchLabels:
      app: myapp # Selects pods with this label
  template:
    metadata:
      labels:
        app: myapp # Labels applied to pods created by this ReplicaSet
    spec:
      containers:
      - name: myapp-container
        image: nginx:latest
        ports:
        - containerPort: 80
```

To deploy this ReplicaSet:

```bash
kubectl apply -f myapp-replicaset.yaml
kubectl get rs myapp-replicaset
kubectl get pods -l app=myapp
```

### Replication Controller
A Replication Controller is an older version of ReplicaSet. It serves the same purpose of ensuring a specific number of pod replicas are running but lacks the more flexible selector capabilities of ReplicaSets. ReplicaSets are now the preferred choice.

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myapp-rc
spec:
  replicas: 2 # Desired number of pods
  selector:
    app: myapp-rc # Legacy selector type
  template:
    metadata:
      labels:
        app: myapp-rc
    spec:
      containers:
      - name: myapp-container-rc
        image: nginx:latest
        ports:
        - containerPort: 80
```

To deploy this Replication Controller:

```bash
kubectl apply -f myapp-rc.yaml
kubectl get rc myapp-rc
kubectl get pods -l app=myapp-rc
```

### Deployment
A Deployment provides declarative updates for Pods and ReplicaSets. It manages the rollout and rollback of applications, making it the most common way to manage stateless applications in Kubernetes. Deployments automatically create and manage ReplicaSets to achieve their desired state.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
  labels:
    app: myapp
spec:
  replicas: 3 # Desired number of pods
  selector:
    matchLabels:
      app: myapp # Selects pods with this label
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp-container
        image: nginx:1.14.2 # Initial image version
        ports:
        - containerPort: 80
  strategy:
    type: RollingUpdate # Specifies update strategy
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
```

To deploy this Deployment:

```bash
kubectl apply -f myapp-deployment.yaml
kubectl get deployment myapp-deployment
kubectl get rs -l app=myapp
kubectl get pods -l app=myapp
```

To update the image (triggers a rolling update):

```bash
kubectl set image deployment/myapp-deployment myapp-container=nginx:1.16.1
kubectl rollout status deployment/myapp-deployment
```

To roll back to a previous revision:

```bash
kubectl rollout undo deployment/myapp-deployment
```

### Comparison: ReplicaSet vs. Deployment

| Feature | ReplicaSet | Deployment |
|---------|------------|------------|
| Primary Purpose | Ensures specified number of pod replicas are running | Manages ReplicaSets for declarative updates (rollouts/rollbacks) |
| Update Strategy | No direct update strategy | Supports RollingUpdate and Recreate strategies |
| Rollback Capability | No inherent rollback | Built-in rollback to previous revisions |
| Management | Manages Pods directly | Manages ReplicaSets, which in turn manage Pods |
| Typical Usage | Rarely used directly; managed by Deployments | Standard way to deploy and manage stateless applications |

---

## Kubernetes Service Types

Services define how to access a set of pods, abstracting away their changing IP addresses.

### ClusterIP Service

**Type:** ClusterIP

**Purpose:** Exposes the service on an internal IP address within the cluster. This makes the service only reachable from within the cluster.

**Use Case:** Internal communication between microservices.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-clusterip-service
spec:
  selector:
    app: myapp # Selects pods with this label
  ports:
    - protocol: TCP
      port: 80 # The port on the service
      targetPort: 8080 # The port on the pod container
  type: ClusterIP
```

To deploy and verify:

```bash
kubectl apply -f myapp-clusterip-service.yaml
kubectl get svc myapp-clusterip-service
# Example: Access from another pod in the cluster via http://myapp-clusterip-service
```

### NodePort Service
**Type:** NodePort

**Purpose:** Exposes the service on a static port on each worker node's IP. External traffic can reach the service through any node's IP and the specified NodePort.

**Use Case:** Exposing services to external traffic in development/testing environments, or where a cloud load balancer is not available.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30007 # Optional: Kubernetes assigns if omitted (range 30000-32767)
  type: NodePort
```

To deploy and verify:

```bash
kubectl apply -f myapp-nodeport-service.yaml
kubectl get svc myapp-nodeport-service
# Access: http://<NodeIP>:30007 from outside the cluster
```

### LoadBalancer Service
**Type:** LoadBalancer

**Purpose:** Exposes the service externally using a cloud provider's load balancer. The cloud provider provisions an external IP address that routes traffic to the service.

**Use Case:** Publicly exposing services in cloud environments like AKS, AWS, GCP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-loadbalancer-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
```

To deploy and verify:

```bash
kubectl apply -f myapp-loadbalancer-service.yaml
kubectl get svc myapp-loadbalancer-service
# The EXTERNAL-IP will be assigned by your cloud provider (e.g., Azure Load Balancer).
# Access: http://<EXTERNAL-IP>:80 from outside the cluster
```

## PersistentVolume (PV) and PersistentVolumeClaim (PVC)
PVs and PVCs provide a mechanism for abstracting storage details, allowing pods to request and use persistent storage.

### PersistentVolume (PV)

A PersistentVolume (PV) is a piece of storage in the cluster that has been provisioned by an administrator. It abstracts the underlying storage infrastructure (e.g., Azure Disk, NFS, iSCSI).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-azure-disk-pv
spec:
  capacity:
    storage: 10Gi # Total storage capacity
  accessModes:
    - ReadWriteOnce # Can be mounted as read-write by a single node
  persistentVolumeReclaimPolicy: Retain # Retain the volume after PVC deletion
  storageClassName: managed-premium # Matches a StorageClass
  azureDisk: # Example for Azure Disk
    kind: Managed
    diskName: myAKSDisk
    diskURI: /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/providers/Microsoft.Compute/disks/<DISK_NAME>
```

To create a PV:

```bash
kubectl apply -f my-azure-disk-pv.yaml
kubectl get pv my-azure-disk-pv
```

### PersistentVolumeClaim (PVC)
A PersistentVolumeClaim (PVC) is a request for storage by a user. It consumes PV resources. When a PVC is created, Kubernetes attempts to find a matching PV based on specified criteria (capacity, access modes, storage class).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-pvc
spec:
  accessModes:
    - ReadWriteOnce # Must match PV's access mode
  resources:
    requests:
      storage: 5Gi # Requested storage size
  storageClassName: managed-premium # Must match PV's storageClassName (or be omitted for default)
```

To create a PVC:

```bash
kubectl apply -f my-app-pvc.yaml
kubectl get pvc my-app-pvc
# Verify its status changes from Pending to Bound
```

### Attaching PVC to a Pod
Once a PVC is bound to a PV, it can be mounted into a pod's containers.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod-with-pvc
spec:
  volumes:
    - name: my-persistent-storage
      persistentVolumeClaim:
        claimName: my-app-pvc # Reference the PVC by name
  containers:
    - name: my-app-container
      image: busybox
      command: ["/bin/sh", "-c", "echo 'Hello from persistent storage' > /mnt/data/hello.txt; sleep 3600"]
      volumeMounts:
        - name: my-persistent-storage
          mountPath: /mnt/data # Path inside the container where the volume is mounted
```

To deploy and verify:

```bash
kubectl apply -f my-pod-with-pvc.yaml
kubectl exec my-pod-with-pvc -- ls /mnt/data/
kubectl exec my-pod-with-pvc -- cat /mnt/data/hello.txt
```

## Managing Kubernetes with Azure Kubernetes Service (AKS)
AKS simplifies the deployment, management, and operations of Kubernetes.

### Creating an AKS Cluster

Use the Azure CLI to create a resource group and then an AKS cluster.

```bash
# Create a resource group
az group create --name myAKSResourceGroup --location eastus

# Create an AKS cluster
az aks create --resource-group myAKSResourceGroup --name myAKSCluster --node-count 1 --generate-ssh-keys --enable-managed-identity
```

### Connecting to an AKS Cluster

After creation, configure kubectl to connect to your new AKS cluster.

```bash
az aks get-credentials --resource-group myAKSResourceGroup --name myAKSCluster
kubectl config view
kubectl get nodes
```

### Scaling an AKS Cluster

Adjust the number of nodes in your AKS cluster.

```bash
# Scale the node count of an existing cluster
az aks scale --resource-group myAKSResourceGroup --name myAKSCluster --node-count 2
```

### Upgrading an AKS Cluster

Upgrade the Kubernetes version of your AKS cluster.

```bash
# Check available upgrade versions
az aks get-upgrades --resource-group myAKSResourceGroup --name myAKSCluster --output table

# Upgrade the cluster to a specific version
az aks upgrade --resource-group myAKSResourceGroup --name myAKSCluster --kubernetes-version 1.29.0
```

## Configure Liveness and Readiness Probes for Pods
Health probes ensure your applications running in pods are healthy and ready to serve traffic.

### Liveness Probe

A liveness probe determines if your container is running. If it fails, Kubernetes restarts the container.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-with-liveness
spec:
  containers:
  - name: myapp-container
    image: nginx
    livenessProbe:
      httpGet: # HTTP GET probe
        path: /healthz
        port: 80
      initialDelaySeconds: 15 # Wait 15 seconds before first probe
      periodSeconds: 20 # Probe every 20 seconds
      timeoutSeconds: 5 # Probe times out after 5 seconds
      failureThreshold: 3 # After 3 failures, restart container
    ports:
    - containerPort: 80
```

### Readiness Probe

A readiness probe determines if your container is ready to serve requests. If it fails, Kubernetes removes the pod's IP address from the service endpoints until it's ready, preventing traffic from being sent to an unready pod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-with-readiness
spec:
  containers:
  - name: myapp-container
    image: nginx
    readinessProbe:
      httpGet: # HTTP GET probe
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      successThreshold: 1 # Number of successes for the probe to be considered successful
    ports:
    - containerPort: 80
```

## Configure Taints and Tolerations
Taints repel pods from a node, while Tolerations allow pods to be scheduled on tainted nodes. They work together to ensure pods are not scheduled onto inappropriate nodes.

### Taints
Apply a taint to a node using kubectl taint. This example taints node1 with key dedicated, value gpu, and effect NoSchedule.

```bash
kubectl taint nodes node1 dedicated=gpu:NoSchedule
```

This means no pods will be scheduled on node1 unless they have a matching toleration.

### Tolerations
Add a toleration to a pod's specification to allow it to be scheduled on a node with a matching taint.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-gpu-pod
spec:
  containers:
  - name: my-gpu-container
    image: nvidia/cuda:11.0-base
  tolerations:
  - key: "dedicated"
    operator: "Equal" # Can also be "Exists"
    value: "gpu"
    effect: "NoSchedule" # Must match the taint's effect
```

To deploy this pod, it will now be able to schedule on node1:

```bash
kubectl apply -f my-gpu-pod.yaml
```

---

## Configure Autoscaling in Your Cluster (Horizontal Scaling)
Horizontal Pod Autoscaling (HPA) automatically scales the number of pods in a deployment or replica set based on observed CPU utilization or other select metrics.

Deploy a sample application:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      app: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m # Define CPU limits for HPA to work effectively
          requests:
            cpu: 200m # Define CPU requests
```

```bash
kubectl apply -f php-apache-deployment.yaml
```

Expose the application with a service:

```bash
kubectl expose deployment php-apache --type=ClusterIP --name=php-apache-service
```

Create a Horizontal Pod Autoscaler: This HPA will target the php-apache deployment, maintaining 1 to 10 replicas, with a target CPU utilization of 50%.

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

Verify HPA creation:

```bash
kubectl get hpa
```

Simulate load to observe scaling (requires a separate pod to generate traffic):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: load-generator
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh", "-c", "while true; do wget -q -O- http://php-apache-service; done"]
```

```bash
kubectl apply -f load-generator.yaml
```

Monitor HPA status: Observe the REPLICAS count increase over time.

```bash
kubectl get hpa php-apache --watch
```

To stop the load generator:

```bash
kubectl delete pod load-generator
```
