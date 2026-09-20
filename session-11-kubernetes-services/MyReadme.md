# Session 11: Kubernetes Services

**Name:** Shaikh Suja Rahaman
**Enrollment Number:** 24bcs10038

---

## SECTION 1: Chronological Task Overview

* **Task 1: Kubernetes Port Architecture & Clarification Drill** — Demystify and document the precise boundaries, scope, and routing path of the 4 Kubernetes ports: `containerPort`, `targetPort`, `port`, and `nodePort`.
* **Task 2: Type 1 Service — ClusterIP (Default Internal Networking)** — Deploy an internal microservice backend, configure a standard ClusterIP Service, verify Endpoints/EndpointSlices, and test internal access via a client pod using service name, virtual IP, and FQDN.
* **Task 3: Type 2 Service — NodePort (Host-Level External Ingress)** — Deploy a web application exposed on a high node port (30000–32767), verify cluster-wide node port binding, and access the application externally via Node IP and browser port-forwarding.
* **Task 4: Type 3 Service — LoadBalancer (Cloud-Native Ingress Simulation)** — Deploy an externally facing service using `type: LoadBalancer`, simulate cloud controller IP allocation using `minikube tunnel`, verify automatic creation of underlying NodePort and ClusterIP, and test public access.
* **Task 5: Type 4 Service — ExternalName (CoreDNS CNAME Alias Redirection)** — Create an ExternalName service pointing to an external domain without selectors or endpoints, and prove CNAME redirection inside a test pod via `nslookup`.
* **Task 6: Type 5 Service — Headless Service (clusterIP: None & Stateful Workloads)** — Deploy a Headless Service paired with a StatefulSet, demonstrate that CoreDNS returns individual Pod IPs (multiple A records) instead of a single virtual IP, and test direct ordinal pod addressing.
* **Task 7: Services Without Selectors (Manual Endpoints Mapping)** — Create a custom ClusterIP service without a label selector, manually construct a matching Endpoints object pointing to an external IP, and demonstrate routing cluster traffic to external infrastructure.
* **Task 8: FQDN & CoreDNS Deep Dive Architecture Analysis** — Investigate the Kubernetes FQDN hierarchical structure, inspect container `/etc/resolv.conf` settings, and document the latency implications of `ndots:5`.
* **Task 9: Pod Identity & Lifecycle Invariance Drill** — Deploy a stateless Deployment alongside a stateful StatefulSet to observe their distinct pod naming schemes. Show that Deployments generate new identities while StatefulSets guarantee invariant ordinal recreation.
* **Task 10: Master Architectural Matrix** — Formulate an exhaustive architectural comparison across the three primary Kubernetes workload controllers (Deployments, StatefulSets, and DaemonSets).
* **Task 11: Production Cost Optimization & Service Selection Decision Tree** — Synthesize an end-to-end Service Selection Decision Tree and analyze the enterprise cloud anti-pattern of provisioning multiple `type: LoadBalancer` services versus a production-grade Ingress pattern.
* **Task 12: Minikube Docker-Driver Port Binding & Tunnel Gotcha Analysis** — Investigate why `<Node-IP>:<NodePort>` fails on macOS/Windows with Minikube Docker driver, and demonstrate standard operational workarounds (`minikube service <svc> --url` and `minikube tunnel`).

---

## SECTION 2: Detailed Workflow, Commands & Screenshot Checklist

### Task 1: Kubernetes Port Architecture & Clarification Drill
**Short Description:** Document and visually map the 4 distinct port definitions in Kubernetes. Illustrate how a packet flows from an external client through the node, into the service, and down to the application process inside the container.

**Manifest Reference:** Concept mapping across all `service.yaml` and `app-deployment.yaml` files.

**Commands to Run:**
```bash
# Inspect port declarations across pod and service
kubectl explain pod.spec.containers.ports.containerPort
kubectl explain service.spec.ports
```

**Architecture Flowchart:**
```
Client Browser ──► [nodePort: 30080] (Host IP)
                        │
                        ▼
                   [port: 8080] (Service VIP)
                        │
                        ▼
                   [targetPort: 80] (Pod Network)
                        │
                        ▼
                   [containerPort: 80] (Container Engine / Nginx)
```

**Screenshot 1.1:**
![Port Architecture Diagram](./screenshots/Screenshot%201.1.png)

---

### Task 2: Type 1 Service — ClusterIP (Default Internal Networking)
**Short Description:** Deploy a 3-replica backend (`web-app-clusterip`), create a ClusterIP service on port 8080 targeting container port 80, inspect automatic endpoint binding, and test connectivity from an ephemeral client pod.

**Working Directory:** `session-11-kubernetes-services/01-clusterip/`

**Commands to Run:**
```bash
# 1. Deploy backend app and ClusterIP service
kubectl apply -f 01-clusterip/app-deployment.yaml
kubectl apply -f 01-clusterip/service.yaml

# 2. Verify pods, service, and endpoints
kubectl get pods -l app=web-clusterip -o wide
kubectl get svc web-service-clusterip
kubectl get endpoints web-service-clusterip

# 3. Deploy diagnostic client pod
kubectl apply -f 01-clusterip/client-pod.yaml
kubectl wait --for=condition=ready pod/curl-client --timeout=60s

# 4. Test internal resolution methods from inside the cluster
kubectl exec -it curl-client -- curl -s http://web-service-clusterip:8080 | grep -i "<title>"
kubectl exec -it curl-client -- curl -s http://web-service-clusterip.default.svc.cluster.local:8080 | grep -i "<title>"
```

**Screenshots:**
![ClusterIP Endpoints Verification](./screenshots/Screenshot%202.1.png)
![ClusterIP Client Resolution](./screenshots/Screenshot%202.2.png)

---

### Task 3: Type 2 Service — NodePort (Host-Level External Ingress)
**Short Description:** Deploy a 2-replica Nginx app and expose it externally by opening port 30080 on every cluster node. Verify that hitting any node IP on port 30080 directs traffic to the underlying pods.

**Working Directory:** `session-11-kubernetes-services/02-nodeport/`

**Commands to Run:**
```bash
# 1. Deploy application and NodePort service
kubectl apply -f 02-nodeport/app-deployment.yaml
kubectl apply -f 02-nodeport/service.yaml

# 2. Verify the NodePort mapping
kubectl get svc web-service-nodeport

# 3. Retrieve Minikube IP and verify node port access
MINIKUBE_IP=$(minikube ip)
curl -I http://${MINIKUBE_IP}:30080

# 4. Alternatively test via minikube service tunnel (macOS/Docker driver)
minikube service web-service-nodeport --url
```

**Screenshots:**
![NodePort Service Mapping](./screenshots/Screenshot%203.1.png)
![NodePort Curl Response](./screenshots/Screenshot%203.2.png)

---

### Task 4: Type 3 Service — LoadBalancer (Cloud-Native Ingress Simulation)
**Short Description:** Deploy a 3-replica workload exposed through `type: LoadBalancer`. Use `minikube tunnel` to simulate a cloud provider assigning an `EXTERNAL-IP`, and confirm that Kubernetes automatically configures internal NodePort and ClusterIP layers.

**Working Directory:** `session-11-kubernetes-services/03-loadbalancer/`

**Commands to Run:**
```bash
# 1. Deploy application and LoadBalancer service
kubectl apply -f 03-loadbalancer/app-deployment.yaml
kubectl apply -f 03-loadbalancer/service.yaml

# 2. Check service status (will initially show <pending> without tunnel)
kubectl get svc web-service-loadbalancer

# 3. In a separate terminal, start the Minikube LoadBalancer tunnel
minikube tunnel

# 4. In primary terminal, observe EXTERNAL-IP populated
kubectl get svc web-service-loadbalancer

# 5. Access the application directly on standard port 80
EXTERNAL_IP=$(kubectl get svc web-service-loadbalancer -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -s http://${EXTERNAL_IP}:80 | grep -i "<title>"
```

**Screenshots:**
![LoadBalancer External IP Populated](./screenshots/Screenshot%204.1.png)
![LoadBalancer Direct Access](./screenshots/Screenshot%204.2.png)

---

### Task 5: Type 4 Service — ExternalName (CoreDNS CNAME Alias Redirection)
**Short Description:** Create an ExternalName service that acts as an internal DNS CNAME alias pointing to an external domain (e.g., api.github.com). Verify that no cluster IP or endpoints are created, and confirm CNAME resolution using `nslookup`.

**Working Directory:** `session-11-kubernetes-services/04-externalname/`

**Commands to Run:**
```bash
# 1. Apply ExternalName service and client pod
kubectl apply -f 04-externalname/service.yaml
kubectl apply -f 04-externalname/client-pod.yaml
kubectl wait --for=condition=ready pod/dns-test-client --timeout=60s

# 2. Inspect the service (Notice CLUSTER-IP is <none>, EXTERNAL-IP is the target domain)
kubectl get svc external-database-service

# 3. Verify DNS resolution returns canonical name (CNAME)
kubectl exec -it dns-test-client -- nslookup external-database-service

# 4. Test outbound traffic through the alias
kubectl exec -it dns-test-client -- curl -s -k https://external-database-service
```

**Screenshots:**
![ExternalName Service Status](./screenshots/Screenshot%205.1.png)
![nslookup ExternalName Resolution](./screenshots/Screenshot%205.2.png)

---

### Task 6: Type 5 Service — Headless Service (clusterIP: None & Stateful Workloads)
**Short Description:** Deploy a 3-replica StatefulSet with a Headless Service (`clusterIP: None`). Prove that CoreDNS returns individual A records for all matching Pod IPs directly rather than a single VIP, and curl an ordinal pod hostname directly.

**Working Directory:** `session-11-kubernetes-services/05-headless/`

**Commands to Run:**
```bash
# 1. Apply Headless service and StatefulSet
kubectl apply -f 05-headless/service.yaml
kubectl apply -f 05-headless/app-statefulset.yaml
kubectl apply -f 05-headless/client-pod.yaml

# 2. Wait for stateful pods (web-stateful-0, 1, 2) to become Ready
kubectl rollout status statefulset/web-stateful --timeout=120s
kubectl get pods -l app=web-headless -o wide

# 3. Inspect Service (Notice CLUSTER-IP is explicitly None)
kubectl get svc web-service-headless

# 4. Perform DNS lookup on Headless Service name -> Returns ALL pod IPs
kubectl exec -it headless-dns-client -- nslookup web-service-headless

# 5. Query an individual Pod directly via its stable FQDN
kubectl exec -it headless-dns-client -- nslookup web-stateful-0.web-service-headless.default.svc.cluster.local
kubectl exec -it headless-dns-client -- curl -s http://web-stateful-0.web-service-headless:80 | grep -i "<title>"
```

**Screenshots:**
![nslookup Headless Service A Records](./screenshots/Screenshot%206.1.png)
![Curl Direct Stateful Pod Hostname](./screenshots/Screenshot%206.2.png)

---

### Task 7: Services Without Selectors (Manual Endpoints Mapping)
**Short Description:** Define a Service without selectors (`empty-endpoints.yaml` pattern) and manually bind it to an external backend IP address using a separate Endpoints manifest. Demonstrate how Kubernetes abstracts legacy or external infrastructure.

**Commands to Run:**
```bash
# 1. Create Service without a selector
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: external-legacy-db
spec:
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
EOF

# 2. Verify endpoints are initially empty (<none>)
kubectl get endpoints external-legacy-db

# 3. Manually create matching Endpoints object pointing to external IP
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Endpoints
metadata:
  name: external-legacy-db
subsets:
  - addresses:
      - ip: 192.168.1.150
    ports:
      - port: 3306
EOF

# 4. Verify endpoints are now successfully attached
kubectl get endpoints external-legacy-db
```

**Screenshots:**
![Service Initial Empty Endpoints](./screenshots/Screenshot%207.1.png)
![Endpoints Mapped to External IP](./screenshots/Screenshot%207.2.png)

---

### Task 8: FQDN & CoreDNS Deep Dive Architecture Analysis
**Short Description:** Inspect the cluster DNS configuration inside running pods. Break down the anatomy of a Kubernetes FQDN, examine `/etc/resolv.conf`, test search domain completion, and explain why `ndots:5` causes latency in production when making external API calls.

**Commands to Run:**
```bash
# 1. Verify CoreDNS pods are active in kube-system
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide

# 2. Inspect /etc/resolv.conf inside any running pod
kubectl exec -it curl-client -- cat /etc/resolv.conf

# 3. Test DNS search domain expansion
kubectl exec -it curl-client -- nslookup web-service-clusterip

# 4. Demonstrate ndots:5 external query latency mechanism
kubectl exec -it curl-client -- nslookup api.github.com
```

**Screenshot:**
![DNS Analysis: resolv.conf & nslookup](./screenshots/Screenshot%208.1%20and%208.2.png)

---

### Task 9: Pod Identity & Lifecycle Invariance Drill
**Short Description:** Deploy a stateless Deployment alongside an ordinal StatefulSet. Inspect their naming schemes, delete a running pod from each controller using `kubectl delete pod`, and observe that the Deployment spawns an ephemeral pod with a completely new random hash, whereas the StatefulSet strictly resurrects the exact same ordinal index (`web-stateful-0`).

**Commands to Run:**
```bash
# 1. Apply both workloads
kubectl apply -f session-11-kubernetes-services/01-clusterip/app-deployment.yaml
kubectl apply -f session-11-kubernetes-services/05-headless/service.yaml
kubectl apply -f session-11-kubernetes-services/05-headless/app-statefulset.yaml

# 2. Wait for pods to become Ready and observe the naming conventions
kubectl get pods -l app=web-clusterip
kubectl get pods -l app=web-headless

# 3. Capture the exact pod names before deletion
DEPLOY_POD=$(kubectl get pods -l app=web-clusterip -o jsonpath='{.items[0].metadata.name}')
echo "Deleting Stateless Deployment Pod: ${DEPLOY_POD}"
kubectl delete pod "${DEPLOY_POD}"

# 4. Check the Deployment pods immediately — notice a brand-new random hash is generated!
kubectl get pods -l app=web-clusterip

# 5. Delete an ordinal StatefulSet pod (web-stateful-0)
echo "Deleting StatefulSet Pod: web-stateful-0"
kubectl delete pod web-stateful-0

# 6. Check the StatefulSet pods immediately — notice web-stateful-0 is recreated identically!
kubectl get pods -l app=web-headless
```

**Screenshot:**
![Pod Identity Stateful vs Stateless](./screenshots/Screenshot%209.1%20and%209.2.png)

---

### Task 10: Master Architectural Matrix — Deployment vs. StatefulSet vs. DaemonSet
**Short Description:** Engineering reference matrix evaluating the differences between Deployments, StatefulSets, and DaemonSets.

| Architectural Metric | Deployment | StatefulSet | DaemonSet |
| :--- | :--- | :--- | :--- |
| **Primary Workload Type** | Stateless microservices, Web APIs | Clustered databases, Distributed queues | Node-level infrastructure agents |
| **Pod Naming Scheme** | Random hash (`<deploy>-<rs-hash>-<random>`) | Deterministic ordinal (`<name>-0, 1, 2`) | Deterministic node hash (`<ds>-<random>`) |
| **Pod Identity Persistence** | Ephemeral (disposable upon death) | Invariant (identity, IP, hostname stick) | Bound to individual worker node |
| **Startup / Shutdown Order** | Non-ordered, parallel | Strictly sequential (0 -> 1 -> 2, reversed on termination) | Parallel across all eligible nodes |
| **Storage Mechanism** | Shared volume or ephemeral `emptyDir` | Dedicated PersistentVolume per ordinal via `volumeClaimTemplates` | HostPath mounts or node-local storage |
| **Associated Service Type** | Standard ClusterIP / NodePort / LoadBalancer | Headless Service (`clusterIP: None`) mandatory for discovery | None or local ClusterIP |
| **Scaling Behavior** | Scales arbitrarily across healthy nodes | Scales ordinally (adds/removes at the tail) | Scales automatically when nodes join/leave |
| **Production Examples** | Nginx, Python Flask, Node.js API, Go services | Kafka, MongoDB, Cassandra, PostgreSQL, ZooKeeper | Fluentd, Prometheus Node Exporter, Cilium, Falco |

**Screenshot:**
![Architectural Matrix](./screenshots/Screenshot%2010.1.png)

---

### Task 11: Production Cost Optimization & Service Selection Decision Tree
**Short Description:** Document the Kubernetes Service Decision Tree and conduct a cost-optimization analysis. Detail why provisioning 50 `type: LoadBalancer` services creates an enterprise billing anti-pattern in public clouds (AWS/GCP/Azure) and demonstrate how an Ingress Controller eliminates this overhead.

**Architecture Diagram:**
```text
ANTI-PATTERN (Expensive: $25/mo per service):
Microservice A ──► AWS NLB 1 ($25/mo) ──► ClusterIP A
Microservice B ──► AWS NLB 2 ($25/mo) ──► ClusterIP B
Microservice C ──► AWS NLB 3 ($25/mo) ──► ClusterIP C
Total for 50 services = $1,250 / month

BEST PRACTICE (Cost-Optimized: Single Entrypoint):
Public Internet ──► 1 Unified AWS Load Balancer ($25/mo)
                            │
                            ▼
                 [ NGINX Ingress Controller ]
                 (Layer 7 Host & Path Routing)
                    │            │            │
                    ▼            ▼            ▼
               ClusterIP A  ClusterIP B  ClusterIP C
Total for 50 services = $25 / month (Savings: $1,225/mo)
```

**Service Selection Logic Tree:**
```text
Need to expose service outside cluster?
│
├── NO ──► Need direct pod-to-pod discovery (Kafka/DB)?
│           ├── YES ──► Use HEADLESS SERVICE (clusterIP: None)
│           └── NO  ──► Use CLUSTERIP (Default)
│
└── YES ──► Connecting to an external 3rd-party domain (AWS RDS / Stripe)?
            ├── YES ──► Use EXTERNALNAME
            └── NO  ──► Are you on Public Cloud (AWS/GCP/Azure)?
                         ├── YES (HTTP/HTTPS) ──► Expose 1 INGRESS via LOADBALANCER,
                         │                        apps as internal CLUSTERIP
                         ├── YES (TCP/UDP)    ──► Direct LOADBALANCER
                         └── NO (On-Prem/Dev) ──► NODEPORT
```

---

### Task 12: Minikube Docker-Driver Port Binding & Tunnel Gotcha Analysis
**Short Description:** Analyze and document why running `curl http://<Node-IP>:<NodePort>` fails on macOS and Windows when using Minikube with the Docker driver. Execute and verify the two standard operational solutions.

**Root Cause Explanation:**
In standard bare-metal Linux clusters, the worker node IP belongs directly to a physical interface reachable on the local network. When using Minikube on macOS or Windows with the Docker driver (`--driver=docker`), Minikube runs inside an isolated Docker container. The node IP (e.g., `192.168.49.2`) belongs to an internal Docker network bridge (`docker0/bridge`) that macOS/Windows host kernels cannot directly route to without specialized proxying.

**Commands to Run & Verify Workarounds:**
```bash
# 1. Re-verify the NodePort service is active
kubectl get svc web-service-nodeport

# 2. Attempt direct curl on Node IP (Demonstrating the failure)
NODE_IP=$(minikube ip)
echo "Testing direct connection to ${NODE_IP}:30080 (Expect timeout/failure on macOS Docker driver)..."
curl --connect-timeout 2 -s http://${NODE_IP}:30080 || echo "Connection Failed as expected!"

# -------------------------------------------------------------
# WORKAROUND 1: Dynamic Local Proxy via Minikube Service
# -------------------------------------------------------------
minikube service web-service-nodeport --url

# -------------------------------------------------------------
# WORKAROUND 2: Continuous L3 Route Tunnel (Production Simulation)
# -------------------------------------------------------------
# In a separate terminal window, launch minikube tunnel
minikube tunnel

# In your primary terminal, test direct localhost access on the mapped port:
curl -I http://localhost:30080
```

**Screenshots:**
![Docker Bridge Network Failure](./screenshots/Screenshot%2012.1.png)
![Minikube Service URL Proxy](./screenshots/Screenshot%2012.2.png)
![Minikube Tunnel Localhost Success](./screenshots/Screenshot%2012.3.png)
