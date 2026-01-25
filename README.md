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

**Environment:** K8s v1.21, Self-managed cluster, Static nodes

**Summary:** A rebooted node failed to rejoin the cluster due to kubelet identity mismatch.

**What Happened:** 
- After a kernel upgrade and reboot, a node didn’t appear in kubectl get nodes. The kubelet logs showed registration issues.

**Diagnosis Steps:**
- Checked system logs and kubelet logs.
- Noticed --hostname-override didn't match the node name registered earlier.
- kubectl get nodes -o wide showed old hostname; new one mismatched due to DHCP/hostname change.

**Root Cause:** Kubelet registered with a hostname that no longer matched its node identity in the cluster.

**Fix / Workaround:**
- Re-joined the node using correct --hostname-override.
- Cleaned up stale node entry from the cluster.

**Lessons Learned:**
- Node identity must remain consistent across reboots.

**How to Avoid:**
- Set static hostnames and IPs.
- Use consistent cloud-init or kubeadm configuration.

---

### 🔹 Scenario #4: Etcd Disk Full Causing API Server Timeout
**Category:** Cluster Management

**Environment:** K8s v1.25, Bare-metal cluster

**Summary:** etcd ran out of disk space, making API server unresponsive.

**What Happened:**
- The cluster started failing API requests. Etcd logs showed disk space errors, and API server logs showed failed storage operations.

**Diagnosis Steps:**
- Used df -h on etcd nodes — confirmed disk full.
- Reviewed /var/lib/etcd – excessive WAL and snapshot files.
- Used etcdctl to assess DB size.

**Root Cause:** Lack of compaction and snapshotting caused disk to fill up with historical revisions and WALs.

**Fix/Workaround:**
- bash
- CopyEdit
- etcdctl compact <rev>
- etcdctl defrag
  - Cleaned logs, snapshots, and increased disk space temporarily.

**Lessons Learned:**
- etcd requires periodic maintenance.

**How to Avoid:**
- Enable automatic compaction.
- Monitor disk space usage of etcd volumes.

---

### 🔹 Scenario #5: Misconfigured Taints Blocking Pod Scheduling
**Category:** Cluster Management

**Environment:** K8s v1.26, Multi-tenant cluster

**Summary:** Critical workloads weren’t getting scheduled due to incorrect node taints.

**What Happened:**
- A user added taints (NoSchedule) to all nodes to isolate their app, but forgot to include tolerations in workloads. Other apps stopped working.

**Diagnosis Steps:**
- Pods stuck in Pending state.
- Used kubectl describe pod <pod> – reason: no nodes match tolerations.
- Inspected node taints via kubectl describe node.

**Root Cause:** Lack of required tolerations on most workloads.

**Fix/Workaround:**
- Removed the inappropriate taints.
- Re-scheduled workloads.

**Lessons Learned:**
- Node taints must be reviewed cluster-wide.

**How to Avoid:**
- Educate teams on node taints and tolerations.
- Restrict RBAC for node mutation.

---

### 🔹 Scenario #6: Kubelet DiskPressure Loop on Large Image Pulls
**Category:** Cluster Management

**Environment:** K8s v1.22, EKS

**Summary:** Continuous pod evictions caused by DiskPressure due to image bloating.

**What Happened:**
- A new container image with many layers was deployed. Node’s disk filled up, triggering kubelet’s DiskPressure condition. Evicted pods created a loop.

**Diagnosis Steps:**
- Checked node conditions: kubectl describe node showed DiskPressure: True.
- Monitored image cache with crictl images.
- Node /var/lib/containerd usage exceeded threshold.

**Root Cause:** Excessive layering in container image and high pull churn caused disk exhaustion.

**Fix/Workaround:**
- Rebuilt image using multistage builds and removed unused layers.
- Increased ephemeral disk space temporarily.

**Lessons Learned:**
- Container image size directly affects node stability.

**How to Avoid:**
- Set resource requests/limits appropriately.
- Use image scanning to reject bloated images.

---

### 🔹 Scenario #7: Node Goes NotReady Due to Clock Skew
**Category:** Cluster Management

**Environment:** K8s v1.20, On-prem

**Summary:** One node dropped from the cluster due to TLS errors from time skew.

**What Happened:** 
- TLS handshakes between the API server and a node started failing. Node became NotReady. Investigation showed NTP daemon was down.

**Diagnosis Steps:**
- Checked logs for TLS errors: “certificate expired or not yet valid”.
- Used timedatectl to check drift – node was 45s behind.
- NTP service was inactive.

**Root Cause:** 
- Large clock skew between node and control plane led to invalid TLS sessions.

**Fix/Workaround:**
- Restarted NTP sync.
- Restarted kubelet after sync.

**Lessons Learned:** 
- Clock sync is critical in TLS-based distributed systems.

**How to Avoid:**
- Use chronyd or systemd-timesyncd.
- Monitor clock skew across nodes.

---

### 🔹 Scenario #8: API Server High Latency Due to Event Flooding
**Category:** Cluster Management

**Environment:** K8s v1.23, Azure AKS

**Summary:** An app spamming Kubernetes events slowed down the entire API server.

**What Happened:** A custom controller logged frequent events (~50/second), causing the etcd event store to choke.

**Diagnosis Steps:**
- Prometheus showed spike in event count.
- kubectl get events --sort-by=.metadata.creationTimestamp showed massive spam.
- Found misbehaving controller repeating failure events.

**Root Cause:** No rate limiting on event creation in controller logic.

**Fix/Workaround:**
- Patched controller to rate-limit record.Eventf.
- Cleaned old events.

**Lessons Learned:** Events are not free – they impact etcd/API server.

**How to Avoid:**
- Use deduplicated or summarized event logic.
- Set API server --event-ttl=1h and --eventRateLimit.

---

### 🔹 Scenario #9: CoreDNS CrashLoop on Startup
**Category:** Cluster Management

**Environment:** K8s v1.24, DigitalOcean

**Summary:** CoreDNS pods kept crashing due to a misconfigured Corefile.

**What Happened:** A team added a custom rewrite rule in the Corefile which had invalid syntax. CoreDNS failed to start.

**Diagnosis Steps:**
- Checked logs: syntax error on startup.
- Used kubectl describe configmap coredns -n kube-system to inspect.
- Reproduced issue in test cluster.

**Root Cause:** Corefile misconfigured – incorrect directive placement.

**Fix/Workaround:**
- Reverted to backup configmap.
- Restarted CoreDNS.

**Lessons Learned:** DNS misconfigurations can cascade quickly.

**How to Avoid:**
- Use a CoreDNS validator before applying config.
- Maintain versioned backups of Corefile.

---

### 🔹 Scenario #10: Control Plane Unavailable After Flannel Misconfiguration
**Category:** Cluster Management

**Environment:** K8s v1.18, On-prem, Flannel CNI

**Summary:** Misaligned pod CIDRs caused overlay misrouting and API server failure.

**What Happened:** A new node was added with a different pod CIDR than what Flannel expected. This broke pod-to-pod and node-to-control-plane communication.

**Diagnosis Steps:**
- kubectl timed out from nodes.
- Logs showed dropped traffic in iptables.
- Compared --pod-cidr in kubelet and Flannel config.

**Root Cause:** Pod CIDRs weren’t consistent across node and Flannel.

**Fix/Workaround:**
- Reconfigured node with proper CIDR range.
- Flushed iptables and restarted Flannel.

**Lessons Learned:** CNI requires strict configuration consistency.

**How to Avoid:**
- Enforce CIDR policy via admission control.
- Validate podCIDR ranges before adding new nodes.

---

### 🔹 Scenario #11: kube-proxy IPTables Rules Overlap Breaking Networking
**Category:** Cluster Management

**Environment:** K8s v1.22, On-prem with kube-proxy in IPTables mode

**Summary:** Services became unreachable due to overlapping custom IPTables rules with kube-proxy rules.

**What Happened:** A system admin added custom IPTables NAT rules for external routing, which inadvertently modified the same chains managed by kube-proxy.

**Diagnosis Steps:**
- DNS and service access failing intermittently.
- Ran iptables-save | grep KUBE- – found modified chains.
- Checked kube-proxy logs: warnings about rule insert failures.

**Root Cause:** Manual IPTables rules conflicted with KUBE-SERVICES chains, causing rule precedence issues.

**Fix/Workaround:**
- Flushed custom rules and reloaded kube-proxy.

bash
```
CopyEdit
iptables -F; systemctl restart kube-proxy
```

**Lessons Learned:** Never mix manual IPTables rules with kube-proxy-managed chains.

**How to Avoid:**
- Use separate IPTables chains or policy routing.
- Document any node-level firewall rules clearly.

---

### 🔹 Scenario #12: Stuck CSR Requests Blocking New Node Joins
**Category:** Cluster Management

**Environment:** K8s v1.20, kubeadm cluster

**Summary:** New nodes couldn’t join due to a backlog of unapproved CSRs.

**What Happened:** A spike in expired certificate renewals caused hundreds of CSRs to queue, none of which were being auto-approved. New nodes waited indefinitely.

**Diagnosis Steps:**
- Ran kubectl get csr – saw >500 pending requests.
- New nodes stuck at kubelet: ```waiting for server signing```.
- Approval controller was disabled due to misconfiguration.
  
**Root Cause:** Auto-approval for CSRs was turned off during a security patch, but not re-enabled.

**Fix/Workaround:**
bash
```
CopyEdit
kubectl certificate approve <csr-name>
```
- Re-enabled the CSR approver controller.

**Lessons Learned:** CSR management is critical for kubelet-node communication.

**How to Avoid:**
- Monitor pending CSRs.
- Don’t disable kube-controller-manager flags like --cluster-signing-cert-file.

---

### 🔹 Scenario #13: Failed Cluster Upgrade Due to Unready Static Pods
**Category:** Cluster Management

**Environment:** K8s v1.21 → v1.23 upgrade, kubeadm

**Summary:** Upgrade failed when static control plane pods weren’t ready due to invalid manifests.

**What Happened:** During upgrade, etcd didn’t come up because its pod manifest had a typo. Kubelet never started etcd, causing control plane install to hang.

**Diagnosis Steps:**
- Checked /etc/kubernetes/manifests/etcd.yaml for errors.
- Used journalctl -u kubelet to see static pod startup errors.
- Verified pod not running via crictl ps.

**Root Cause:** Human error editing the static pod manifest – invalid volumeMount path.

**Fix/Workaround:**
- Fixed manifest.
- Restarted kubelet to load corrected pod.
  
**Lessons Learned:** Static pods need strict validation.

**How to Avoid:**
- Use YAML linter on static manifests.
- Backup manifests before upgrade.

---

### 🔹 Scenario #14: Uncontrolled Logs Filled Disk on All Nodes
**Category:** Cluster Management 

**Environment:** K8s v1.24, AWS EKS, containerd

**Summary:** Application pods generated excessive logs, filling up node /var/log.

**What Happened:** A debug flag was accidentally enabled in a backend pod, logging hundreds of lines/sec. The journald and container logs filled up all disk space.

**Diagnosis Steps:**
- df -h showed /var/log full.
- Checked /var/log/containers/ – massive logs for one pod.
- Used kubectl logs to confirm excessive output.

**Root Cause:** A log level misconfiguration caused explosive growth in logs.

**Fix/Workaround:**
- Rotated and truncated logs.
- Restarted container runtime after cleanup.
- Disabled debug logging.
 
**Lessons Learned:** Logging should be controlled and bounded.

**How to Avoid:**
- Set log rotation policies for container runtimes.
- Enforce sane log levels via CI/CD validation.

---

### 🔹 Scenario #15: Node Drain Fails Due to PodDisruptionBudget Deadlock
**Category:** Cluster Management

**Environment:** K8s v1.21, production cluster with HPA and PDB

**Summary:** kubectl drain never completed because PDBs blocked eviction.

**What Happened:** A deployment had minAvailable: 2 in PDB, but only 2 pods were running. Node drain couldn’t evict either pod without violating PDB.

**Diagnosis Steps:**
- Ran kubectl describe pdb <name> – saw AllowedDisruptions: 0.
- Checked deployment and replica count.
- Tried drain – stuck on pod eviction for 10+ minutes.
  
**Root Cause:** PDB guarantees clashed with under-scaled deployment.

**Fix/Workaround:**
- Temporarily edited PDB to reduce minAvailable.
- Scaled up replicas before drain.
  
**Lessons Learned:** PDBs require careful coordination with replica count.

**How to Avoid:**
- Validate PDBs during deployment scale-downs.
- Create alerts for PDB blocking evictions.

---

### 🔹 Scenario #16: CrashLoop of Kube-Controller-Manager on Boot
**Category:** Cluster Management

**Environment:** K8s v1.23, self-hosted control plane

**Summary:** Controller-manager crashed on startup due to outdated admission controller configuration.

**What Happened:** After an upgrade, the --enable-admission-plugins flag included a deprecated plugin, causing crash.

**Diagnosis Steps:**
- Checked pod logs in /var/log/pods/.
- Saw panic error: “unknown admission plugin”.
- Compared plugin list with K8s documentation.

**Root Cause:** Version mismatch between config and actual controller-manager binary.

**Fix/Workaround:**
- Removed the deprecated plugin from startup flags.
- Restarted pod.

**Lessons Learned:** Admission plugin deprecations are silent but fatal.

**How to Avoid:**
- Track deprecations in each Kubernetes version.
- Automate validation of startup flags.

---

### 🔹 Scenario #17: Inconsistent Cluster State After Partial Backup Restore
**Category:** Cluster Management

**Environment:** K8s v1.24, Velero-based etcd backup

**Summary:** A partial etcd restore led to stale object references and broken dependencies.

**What Happened:** etcd snapshot was restored, but PVCs and secrets weren’t included. Many pods failed to mount or pull secrets.

**Diagnosis Steps:**
- Pods failed with “volume not found” and “secret missing”.
- kubectl get pvc --all-namespaces returned empty.
- Compared resource counts pre- and post-restore.

**Root Cause:** Restore did not include volume snapshots or Kubernetes secrets, leading to an incomplete object graph.

**Fix/Workaround:**
- Manually recreated PVCs and secrets using backups from another tool.
- Redeployed apps.

**Lessons Learned:** etcd backup is not enough alone.

**How to Avoid:**
- Use backup tools that support volume + etcd (e.g., Velero with restic).
- Periodically test full cluster restores.

---

### 🔹 Scenario #18: kubelet Unable to Pull Images Due to Proxy Misconfig
**Category:** Cluster Management

**Environment:** K8s v1.25, Corporate proxy network

**Summary:** Nodes failed to pull images from DockerHub due to incorrect proxy environment configuration.

**What Happened:** New kubelet config missed NO_PROXY=10.0.0.0/8,kubernetes.default.svc, causing internal DNS failures and image pull errors.

**Diagnosis Steps:**
- kubectl describe pod showed ImagePullBackOff.
- Checked environment variables for kubelet via systemctl show kubelet.
- Verified lack of NO_PROXY.

**Root Cause:** Proxy config caused kubelet to route internal cluster DNS and registry traffic through the proxy.

**Fix/Workaround:**
- Updated kubelet service file to include proper NO_PROXY.
- Restarted kubelet.

**Lessons Learned:** Proxies in K8s require deep planning.

**How to Avoid:**
- Always set NO_PROXY with service CIDRs and cluster domains.
- Test image pulls with isolated nodes first.

---

### 🔹 Scenario #19: Multiple Nodes Marked Unreachable Due to Flaky Network Interface
**Category:** Cluster Management

**Environment:** K8s v1.22, Bare-metal, bonded NICs

**Summary:** Flapping interface on switch caused nodes to be marked NotReady intermittently.

**What Happened:** A network switch port had flapping issues, leading to periodic loss of node heartbeats.

**Diagnosis Steps:**
- Node status flapped between Ready and NotReady.
- Checked NIC logs via dmesg and ethtool.
- Observed link flaps in switch logs.

**Root Cause:** Hardware or cable issue causing loss of connectivity.

**Fix/Workaround:**
- Replaced cable and switch port.
- Set up redundant bonding with failover.

**Lessons Learned:** Physical layer issues can appear as node flakiness.

**How to Avoid:**
- Monitor NIC link status and configure bonding.
- Proactively audit switch port health.

---

### 🔹 Scenario #20: Node Labels Accidentally Overwritten by DaemonSet
**Category:** Cluster Management

**Environment:** K8s v1.24, DaemonSet-based node config

**Summary:** A DaemonSet used for node labeling overwrote existing labels used by schedulers.

**What Happened:** A platform team deployed a DaemonSet that set node labels like zone=us-east, but it overwrote custom labels like gpu=true.

**Diagnosis Steps:**
- Pods no longer scheduled to GPU nodes.
- kubectl get nodes --show-labels showed gpu label missing.
- Checked DaemonSet script – labels were overwritten, not merged.

**Root Cause:** Label management script used kubectl label node <node> key=value --overwrite, removing other labels.

**Fix/Workaround:**
- Restored original labels from backup.
- Updated script to merge labels.

**Lessons Learned:** Node labels are critical for scheduling decisions.

**How to Avoid:**
- Use label merging logic (e.g., fetch current labels, then patch).
- Protect key node labels via admission controllers.

---

### 🔹 Scenario #21: Cluster Autoscaler Continuously Spawning and Deleting Nodes
**Category:** Cluster Management

**Environment:** K8s v1.24, AWS EKS with Cluster Autoscaler

**Summary:** The cluster was rapidly scaling up and down, creating instability in workloads.

**What Happened:** A misconfigured deployment had a readiness probe that failed intermittently, making pods seem unready. Cluster Autoscaler detected these as unschedulable, triggering new node provisioning. Once the pod appeared healthy again, Autoscaler would scale down.

**Diagnosis Steps:**
- Monitored Cluster Autoscaler logs (kubectl -n kube-system logs -l app=cluster-autoscaler).
- Identified repeated scale-up and scale-down messages.
- Traced back to a specific deployment’s readiness probe.

**Root Cause:** Flaky readiness probe created false unschedulable pods.

**Fix/Workaround:**
- Fixed the readiness probe to accurately reflect pod health.
- Tuned scale-down-delay-after-add and scale-down-unneeded-time settings.

**Lessons Learned:** Readiness probes directly impact Autoscaler decisions.

**How to Avoid:**
- Validate all probes before production deployments.
- Use Autoscaler logging to audit scaling activity.

---

### 🔹 Scenario #22: Stale Finalizers Preventing Namespace Deletion
**Category:** Cluster Management

**Environment:** K8s v1.21, self-managed

**Summary:** A namespace remained in “Terminating” state indefinitely.

**What Happened:** The namespace contained resources with finalizers pointing to a deleted controller. Kubernetes waited forever for the finalizer to complete cleanup.

**Diagnosis Steps:**
- Ran kubectl get ns <name> -o json – saw dangling finalizers.
- Checked for the corresponding CRD/controller – it was uninstalled.

**Root Cause:** Finalizers without owning controller cause resource lifecycle deadlocks.

**Fix/Workaround:**
- Manually removed finalizers using a patched JSON:

bash
```
CopyEdit
kubectl patch ns <name> -p '{"spec":{"finalizers":[]}}' --type=merge
```

**Lessons Learned:** Always delete CRs before removing the CRD or controller.

**How to Avoid:**
- Implement controller cleanup logic.
- Audit finalizers periodically.
