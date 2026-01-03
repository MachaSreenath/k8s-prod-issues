# 🐳 Kubernetes Production Issues

Welcome to the **Kubernetes Production Issues** repository!  
This repo documents **real-world Kubernetes production problems** and how they were diagnosed, fixed, and prevented. It’s meant as a **cheat sheet for DevOps engineers and K8s admins**.  

> 💡 Pro Tip: Always test fixes in a staging cluster before applying them to production.

---

## 📌 Table of Scenarios

| # | Scenario | Environment | Key Takeaway |
|---|---------|------------|--------------|
| 1 | Zombie Pods Blocking Node Drain | K8s v1.23 On-prem | Monitor Terminating pods & avoid unnecessary finalizers |
| 2 | API Server Crash Due to Excessive CRD Writes | K8s v1.24 GKE | Guard reconcile loops & add CR count alerts |
| 3 | Node Not Rejoining After Reboot | K8s v1.21 Self-managed | Keep kubelet identity consistent |
| 4 | Etcd Disk Full Causing API Timeout | K8s v1.25 Bare-metal | Maintain etcd compaction & monitor disk usage |
| 5 | Misconfigured Taints Blocking Pods | K8s v1.26 Multi-tenant | Review taints/tolerations before applying |
| 6 | Kubelet DiskPressure Loop on Large Images | K8s v1.22 EKS | Optimize image size & set node disk limits |
| 7 | Node NotReady Due to Clock Skew | K8s v1.20 On-prem | Keep clocks in sync across nodes |
| 8 | API Server High Latency from Event Flooding | K8s v1.23 AKS | Rate-limit event generation |
| 9 | CoreDNS CrashLoop on Startup | K8s v1.24 DigitalOcean | Validate Corefile configs & keep backups |
| 10 | Control Plane Unavailable After Flannel Misconfig | K8s v1.18 On-prem | Ensure consistent pod CIDRs & CNI configs |

---

## 📘 Scenarios in Detail

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
