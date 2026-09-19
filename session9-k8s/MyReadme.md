# Session 9: Kubernetes Fundamentals & Cluster Architecture
**Author:** Suja Rahaman
**Course:** SST DevOps & Cloud [SWE]
**Session:** 09 - Kubernetes Fundamentals
**Repository:** devops-heros / session9-k8s

---

## Task 1 & 2: Minikube Installation, Setup & Start
Verify that Minikube and the Kubernetes CLI (`kubectl`) are successfully installed on the local system, and initialize the local single-node Kubernetes cluster using the containerized runtime environment.

**Commands:**
```bash
minikube version
kubectl version --client
minikube start
```

**Output:**
```
minikube version: v1.33.1 commit: e32c234d1081dc36b5c3b10b0a0714b9b9886ac0
Client Version: v1.30.2 Kustomize Version: v5.0.4-0.20230601165947-6ce0bd390ce3

😄  minikube v1.33.1 on Darwin 14.5 (arm64)
✨  Automatically selected the docker driver. Other choices: qemu2, ssh
📌  Using Docker Desktop driver with root permissions
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.44 ...
🔥  Creating docker container (CPUs=2, Memory=4000MB) ...
🐳  Preparing Kubernetes v1.30.0 on containerd 1.7.15 ...
    ▪ Generating certificates and keys ...
    ▪ Booting up control plane ...
    ▪ Configuring RBAC rules ...
🔗  Configuring bridge CNI (Container Network Interface) ...
🔎  Verifying Kubernetes components...
    ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

**Screenshot:**
![Installation, Setup and Start](./screenshots/installation%20and%20setup.png)

---

## Task 3: Verifying Cluster Status & Node Health
Inspect the status of the local cluster control plane, kubelet, API server, and verify the node is in Ready state.

**Commands:**
```bash
minikube status
kubectl get nodes -o wide
```

**Output:**
```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

NAME       STATUS   ROLES           AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
minikube   Ready    control-plane   2m15s   v1.30.0   192.168.49.2   <none>        Ubuntu 22.04.4 LTS   6.6.137+rpt-rpi-v8  containerd://1.7.15
```

**Screenshot:**
![Minikube Status and Nodes](./screenshots/Minikube%20Status%20and%20Nodes.png)

---

## Task 4: Stopping the Minikube Cluster
Gracefully power down the Minikube cluster VM/container to release system resources.

**Command:**
```bash
minikube stop
minikube status
```

**Output:**
```
✋  Stopping node "minikube" ...
🛑  Powering off "minikube" via SSH ...
🛑  1 node stopped.

minikube
type: Control Plane
host: Stopped
kubelet: Stopped
apiserver: Stopped
kubeconfig: Configured
```

**Screenshot:**
![Minikube Stop](./screenshots/Minikube%20Stop.png)

---

## Task 5: Kubernetes Cluster Architecture & Component Analysis
Comprehensive breakdown of the core components powering a Kubernetes cluster based on official documentation and classroom discussion.

### 1. Control Plane (Master Node) Components
* **kube-apiserver (The Front Door):** Acts as the single entry point for all administrative tasks and internal communications. Exposes the Kubernetes HTTP/JSON REST API. Every command (kubectl, web dashboard, internal controllers) must authenticate and communicate through the API server. No component directly accesses etcd except the API server.
* **etcd (The Brain & State Storage):** A distributed, highly available, consistent key-value store. Stores the entire cluster state, specifications, secrets, and metadata.
* **kube-scheduler (The Placement Engine):** Continuously watches for newly created Pods that have no assigned worker node. Analyzes resource requirements and constraints to pick the optimal worker node to run the Pod.
* **kube-controller-manager (The Enforcer / Reconciliation Loop):** Executes continuous control loops that check: Current State == Desired State. Contains sub-controllers such as the Node Controller and ReplicaSet Controller.

### 2. Worker Node (Data Plane) Components
* **kubelet (The Node Captain):** The primary agent running on every worker node. Receives PodSpec objects from kube-apiserver and instructs the Container Runtime to pull images and start containers. Continuously monitors container health.
* **kube-proxy (The Network Router):** Network proxy running on each node that maintains network rules. Enables Kubernetes Services to route TCP/UDP packets across pods, handling internal cluster routing and load balancing.
* **Container Runtime Interface (CRI):** The software responsible for actually running containers (e.g., containerd or CRI-O).
* **Pod (The Smallest Deployable Unit):** The fundamental unit of execution in Kubernetes. Encapsulates one or more tightly coupled containers sharing the same network namespace and storage volumes.
