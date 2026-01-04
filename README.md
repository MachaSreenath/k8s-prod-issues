# 🐳 Kubernetes Production Issues

Welcome to the **Kubernetes Production Issues** repository!  
This repo documents **real-world Kubernetes production problems** and how they were diagnosed, fixed, and prevented. It’s meant as a **cheat sheet for DevOps engineers and K8s admins**.  

> 💡 Pro Tip: Always test fixes in a staging cluster before applying them to production.

---

### 🔹 Scenario #1: Zombie Pods Blocking Node Drain
**Category:** Cluster Management  
**Environment:** K8s v1.23, On-prem bare metal, Systemd cgroups  

**Summary:** Node drain stuck indefinitely due to unresponsive pod with a finalizer.  

**What Happened:**  
- Pod had a **custom finalizer** that never completed termination.  
- API server kept waiting; pod stuck in **Terminating** state.

**Diagnosis Steps:**  
- `kubectl get pods --all-namespaces -o wide` → found stuck pod.  
- `kubectl describe pod <pod>` → saw custom finalizer.  
- Controller managing finalizer had **crashed**.

**Root Cause:** Finalizer logic never executed.  

**Fix / Workaround:**  
```bash
kubectl patch pod <pod-name> -p '{"metadata":{"finalizers":[]}}' --type=merge
```
---

### 🔹 Scenario #2: API Server Crash Due to Excessive CRD Writes
**Category:** Cluster Management  
**Environment:** K8s v1.24, GKE  

**Summary:** API server crashed because a malfunctioning controller flooded the cluster with custom resources.  

**What Happened:**  
- A bug in a controller caused it to create **thousands of Custom Resources (CRs)** in a tight reconciliation loop.  
- Etcd became overloaded → slow writes → API server eventually became unresponsive.  

**Diagnosis Steps:**  
- API latency increased → frequent **504 Gateway Timeout** errors in `kubectl`.  
- `kubectl get crds | wc -l` → huge number of CRs present.  
- Controller logs → infinite reconcile loop on a specific CR type.  
- Etcd disk I/O maxed → database operations slowed down.

**Root Cause:** Bad reconcile logic: the controller **always called create**, regardless of resource state, flooding the cluster with CRs.  

**Fix / Workaround:**  
- Scale the malfunctioning controller to 0 replicas.  
- Manually delete thousands of stale CRs in batches.

**Lessons Learned / Tips:**  
- Always test reconcile logic in a sandbox environment before deploying to production.  
- Implement **create/update guards** in reconciliation loops.  
- Add **Prometheus alerts** for unusually high CR counts.  

**Pro Tip:** Consider implementing **rate limiting** or **circuit breakers** in controllers to prevent uncontrolled resource creation.

---

### 🔹 Scenario #3: Node Not Rejoining After Reboot
**Category:** Cluster Management  
**Environment:** K8s v1.21, Self-managed cluster  

**Summary:** A node failed to rejoin the cluster after a reboot due to a kubelet identity mismatch.

**What Happened:**  
- The node was rebooted after a kernel upgrade.  
- After reboot, the node did **not appear** in `kubectl get nodes`.  
- Kubelet kept failing to register with the control plane.

**Diagnosis Steps:**  
- Checked system and kubelet logs using:
  ```bash
  journalctl -u kubelet
