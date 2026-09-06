# CKA Study Guide — Proxmox Home Lab Edition

*Built for the current exam: Kubernetes **v1.35**, post-2025 curriculum revision. Verified against Linux Foundation docs, August 2026.*

---

## 1. Exam Snapshot

| | |
|---|---|
| **Format** | Performance-based. Live clusters, command line. No multiple choice. |
| **Tasks** | 15–20 (simulator uses 17) |
| **Time** | 2 hours |
| **Passing score** | 66% — partial credit exists |
| **Kubernetes version** | v1.35 (tracks latest minor within 4–8 weeks of release) |
| **Cost** | $445, includes **two exam attempts** and **two Killer.sh simulator sessions** |
| **Eligibility window** | 12 months to schedule and sit |
| **Cert validity** | 2 years |
| **Proctoring** | PSI Secure Browser, remote proctored, **single monitor only** |

### Domain weights

| Domain | Weight | What it actually means |
|---|---|---|
| **Troubleshooting** | **30%** | Broken nodes, broken control plane components, broken services/DNS, resource monitoring, container logs |
| **Cluster Architecture, Installation & Configuration** | **25%** | RBAC, kubeadm bootstrap/join/upgrade, HA control plane, Helm, Kustomize, CNI/CSI/CRI, CRDs & operators |
| **Services & Networking** | **20%** | Pod connectivity, NetworkPolicy, Service types & endpoints, **Gateway API**, Ingress, CoreDNS |
| **Workloads & Scheduling** | **15%** | Rolling updates/rollbacks, ConfigMaps/Secrets, autoscaling (HPA), self-healing primitives, limits & affinity |
| **Storage** | **10%** | StorageClasses, dynamic provisioning, volume types, access modes, reclaim policies, PV/PVC |

> **Read this twice:** Troubleshooting is 30%. Give it a full third of your time. Most people over-study "create a Deployment" and under-study "the kubelet won't start, fix it."

### What changed in the 2025 revision (and why old guides will fail you)

The exam was reworked in Feb 2025. Guides written before then are missing roughly half the new material:

- **Helm and Kustomize** — installing cluster components
- **Gateway API** — GatewayClass, Gateway, HTTPRoute (alongside, not replacing, Ingress)
- **CRDs and operators** — install and configure them
- **Extension interfaces** — conceptual understanding of CNI, CSI, CRI
- **Workload autoscaling** — HPA
- **Dynamic volume provisioning** — StorageClasses, not just static PVs
- **Pod admission and scheduling**

Candidates who've taken it recently report new-curriculum material making up a large share of the questions. Budget real time for these.

### Exam environment mechanics (new-style, important)

The exam is now **SSH-based**. You start on a base node and each task tells you which host to `ssh` into.

- `ssh <nodename>` per the infobox on each task. **Nested SSH is not supported.** `exit` back to base when done.
- `sudo -i` for elevated privileges on any host.
- Pre-installed on each SSH host: `kubectl` with a `k` alias and bash completion, `yq`, `curl`, `wget`, `man`.
- The base host has **none** of these tools — all work happens on the designated host.
- Terminal copy/paste is `Ctrl+Shift+C` / `Ctrl+Shift+V`. Use `Ctrl+Alt+W`, not `Ctrl+W`. The `Insert` key is disabled (use `i` in vim).

**Practice the SSH-hop habit.** A depressing number of points are lost by running a command on the wrong node.

### Documentation allowed during the exam

Only these. Bookmark and *practice navigating them*:

- `https://kubernetes.io/docs` (site search allowed; you must not open external search results)
- `https://kubernetes.io/blog/`
- `https://helm.sh/docs`
- `https://gateway-api.sigs.k8s.io` — **CKA only**
- Task-specific links in the Quick Reference box

---

## 2. Phase 0 — Build the Proxmox Lab (Week 0)

Your Proxmox box is a genuine advantage over people practicing on managed clusters or Killercoda alone. Managed clusters hide the control plane; the CKA tests the control plane. And **snapshots let you break things fearlessly**, which is the single best way to train the 30% troubleshooting domain.

### Topology

**Primary cluster (build this first, keep it forever):**

| VM | Role | vCPU | RAM | Disk |
|---|---|---|---|---|
| `cka-cp1` | control plane | 2 | 4 GB | 30 GB |
| `cka-w1` | worker | 2 | 2–4 GB | 30 GB |
| `cka-w2` | worker | 2 | 2–4 GB | 30 GB |

That's ~10 GB RAM. Add a 4th node later purely as a "join this broken node" target.

**HA cluster (build once around Week 6, then delete):** 3 control planes + 2 workers + 1 small HAProxy/keepalived VM. ~16–20 GB RAM. You only need to do this once to understand it.

**Optional:** an NFS server VM (1 vCPU, 1 GB) for ReadWriteMany and reclaim-policy practice.

### Build the golden template

On the Proxmox host:

```bash
cd /var/lib/vz/template/iso
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

qm create 9000 --name ubuntu-2404-k8s --memory 4096 --cores 2 \
  --net0 virtio,bridge=vmbr0 --cpu host
qm importdisk 9000 noble-server-cloudimg-amd64.img local-lvm
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --boot c --bootdisk scsi0
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1
qm resize 9000 scsi0 +25G
qm template 9000
```

Clone and assign static IPs:

```bash
qm clone 9000 201 --name cka-cp1 --full
qm set 201 --ipconfig0 ip=192.168.1.201/24,gw=192.168.1.1 \
  --ciuser ubuntu --sshkeys ~/.ssh/id_rsa.pub
qm set 201 --memory 4096 --cores 2
qm start 201
# repeat for 202 (cka-w1), 203 (cka-w2)
```

### Proxmox gotchas that will waste your evening

1. **Duplicate machine-id / product_uuid.** Cloned VMs share these and `kubeadm` will refuse to join. Before converting to a template, run inside the VM:
   ```bash
   truncate -s 0 /etc/machine-id
   rm -f /var/lib/dbus/machine-id
   ln -s /etc/machine-id /var/lib/dbus/machine-id
   ```
   Verify after cloning: `sudo cat /sys/class/dmi/id/product_uuid` must differ per node.
2. **Set CPU type to `host`.** Default `kvm64` is noticeably slower and occasionally trips container runtime checks.
3. **Disable memory ballooning** (or set minimum = maximum). The kubelet gets very unhappy when RAM is pulled out from under it and you'll waste hours chasing a phantom problem.
4. **Install `qemu-guest-agent`** in the template so Proxmox can cleanly shut down VMs for snapshots.
5. **Swap.** Cloud images may enable zram or a swapfile. `swapoff -a` plus commenting the fstab entry, and check `systemctl list-units | grep -i swap`.

### Node preparation (all nodes)

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# containerd
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

**Install v1.34, not v1.35.** Then your first real lab task is upgrading to v1.35 — which is an exam topic you'd otherwise never practice.

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Bootstrap:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.1.201
# Calico (has full NetworkPolicy support — Flannel does not, and you need it)
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/master/manifests/calico.yaml
```

### Add-ons you need for full curriculum coverage

| Add-on | Why you need it | Note |
|---|---|---|
| **metrics-server** | `kubectl top`, HPA | Patch with `--kubelet-insecure-tls` in a lab |
| **MetalLB** | `type: LoadBalancer` services actually get an IP | Give it a small pool from your LAN subnet |
| **ingress-nginx** | Ingress resources | Pair with MetalLB |
| **Gateway API CRDs + a controller** | New exam topic | NGINX Gateway Fabric is the easiest bare-metal option |
| **local-path-provisioner** | Dynamic provisioning, StorageClasses | Rancher's; two-minute install |
| **NFS server VM + nfs-subdir provisioner** | ReadWriteMany, reclaim policies | Optional but worth it |
| **Helm** | Exam topic | Install charts, template them, roll back releases |

### The snapshot workflow — your secret weapon

Once the cluster is healthy with add-ons installed:

```bash
# On the Proxmox host — shut down ALL nodes first for a consistent snapshot
for id in 201 202 203; do qm shutdown $id; done
# wait for all to stop, then:
for id in 201 202 203; do qm snapshot $id clean-v134 --description "healthy cluster, pre-break"; done
for id in 201 202 203; do qm start $id; done
```

To restore after you've wrecked something:

```bash
for id in 201 202 203; do qm stop $id; qm rollback $id clean-v134; qm start $id; done
```

**Always snapshot and restore all nodes together.** Restoring a single node into a cluster whose etcd has moved on gives you certificate and clock-skew errors that teach you nothing useful. After any rollback, give the cluster 2–3 minutes and check `kubectl get nodes` before concluding something is broken.

Take a fresh snapshot at each milestone: `clean-v134`, `clean-v135-upgraded`, `with-gateway-api`, and so on.

---

## 3. The 10-Week Plan

Assumes **8–10 hours/week**. Compress to 6 weeks at 15+ hrs/week if you already run Kubernetes at work; stretch to 14 if you're new to containers.

**The 70/30 rule: at least 70% of every session is hands-on in a terminal.** Watching videos feels like progress and isn't. If you finish a study session without having typed `kubectl` fifty times, that session didn't count.

### Week 0 — Lab build
Everything in Phase 0. End state: 3-node cluster on v1.34, Calico, snapshot taken. Also install `kubectl` locally and get your `.bashrc` and `.vimrc` set up (see §5).

### Week 1 — Architecture & kubectl fluency
- Control plane components: what api-server, scheduler, controller-manager, etcd, kubelet, kube-proxy each actually do
- Static pods in `/etc/kubernetes/manifests` — how they differ from normal pods, and that you never `apply` them
- `kubectl` verbs, `-o wide`, `-o yaml`, `-o jsonpath`, `--dry-run=client -o yaml`, `explain`, `describe`, `api-resources`
- Contexts and namespaces: `kubectl config use-context`, `set-context --current --namespace=x`
- **Drill:** recreate every core resource from imperative commands only. No copy-paste from docs.

### Week 2 — Workloads & Scheduling (15%)
- Deployments: rolling update strategy, `maxSurge`/`maxUnavailable`, `rollout status/history/undo`, `--record` is deprecated — use annotations
- ReplicaSets, DaemonSets, StatefulSets, Jobs, CronJobs
- Probes: liveness, readiness, startup — and what each one actually causes to happen
- ConfigMaps and Secrets: as env vars, as volumes, `envFrom`, and what changes on update
- Requests, limits, QoS classes, LimitRange, ResourceQuota
- Scheduling: `nodeSelector`, node affinity/anti-affinity, pod affinity/anti-affinity, taints and tolerations, `nodeName`, topology spread constraints
- **HPA** — `kubectl autoscale`, and writing the manifest by hand. Needs metrics-server.
- **Drill:** deploy an app, scale it with HPA under `stress` load, watch it scale back down.

### Week 3 — Services, Endpoints, CoreDNS (part 1 of 20%)
- ClusterIP, NodePort, LoadBalancer, ExternalName, headless services
- `kubectl get endpoints` / `endpointslices` — **this is your #1 service debugging tool**
- Selector vs. label mismatches; `port` vs `targetPort` vs `nodePort`
- CoreDNS: the Corefile ConfigMap, DNS naming (`svc.namespace.svc.cluster.local`), resolving from inside a pod
- `kube-proxy` modes, briefly
- **Drill:** create a service with a deliberately wrong selector. Diagnose it from symptoms alone in under 3 minutes.

### Week 4 — Ingress, Gateway API, NetworkPolicy (part 2 of 20%)
- Ingress resources, `ingressClassName`, path types, host rules, TLS
- **Gateway API**: GatewayClass → Gateway → HTTPRoute. Understand the role separation (infra provider / cluster operator / app developer) — it's the design's whole point and questions lean on it. Docs at `gateway-api.sigs.k8s.io` are allowed in the exam; get familiar with their layout now.
- NetworkPolicy: default-deny ingress and egress, `podSelector`, `namespaceSelector`, `ipBlock`, port rules, and the additive/OR semantics of multiple policies
- **Drill:** default-deny a namespace, then re-open exactly one path. Verify with `kubectl exec ... -- wget -O- --timeout=2`.

### Week 5 — Storage (10%)
- PV, PVC, binding, `storageClassName` matching
- Access modes: RWO, ROX, RWX, RWOP — and which backends support which
- Reclaim policies: Retain, Delete, Recycle(deprecated) — observe the actual behavior on PV deletion
- StorageClasses and **dynamic provisioning**; default StorageClass annotation
- `volumeMounts`, `emptyDir`, `hostPath`, `subPath`
- CSI conceptually: what a CSI driver is and where it fits
- **Drill:** create a StorageClass, a PVC that binds dynamically, delete the PVC, and predict what happens to the PV before you look.

### Week 6 — Cluster lifecycle (part 1 of 25%)
- `kubeadm init`, `kubeadm join`, `kubeadm token create --print-join-command`
- **`kubeadm upgrade`**: v1.34 → v1.35 on your lab. Full sequence: upgrade plan, upgrade apply on CP, drain, upgrade kubelet+kubectl, uncordon, then each worker. Do it twice.
- Version skew policy — which components can lag which
- `kubectl drain`, `cordon`, `uncordon`, and why `--ignore-daemonsets` is always needed
- **etcd backup and restore** with `etcdctl` — the cert paths are long, this is the classic "bookmark the doc" task. Practice until you can do it from memory-ish.
- **HA control plane**: build the 3-CP + HAProxy cluster once. Stacked vs. external etcd.
- **Drill:** snapshot etcd, delete a bunch of resources, restore, confirm they're back.

### Week 7 — RBAC, Helm, Kustomize, CRDs (part 2 of 25%)
- Role, ClusterRole, RoleBinding, ClusterRoleBinding — and exactly when you need each combination
- ServiceAccounts, token mounting, `kubectl auth can-i --as=system:serviceaccount:ns:name`
- **Helm**: `repo add`, `install`, `upgrade`, `rollback`, `list`, `uninstall`, `template`, `--set` and `-f values.yaml`, `--dry-run`. Install ingress-nginx or metrics-server via Helm to make it real.
- **Kustomize**: `kustomization.yaml`, bases and overlays, `kubectl apply -k`, patches, `commonLabels`, `namePrefix`, ConfigMap generators
- **CRDs and operators**: install a CRD, inspect it with `kubectl get crd` and `kubectl explain`, install an operator (cert-manager or the Prometheus operator are good practice targets), create a custom resource
- **Extension interfaces**: CNI, CSI, CRI — what each is, where the config lives (`/etc/cni/net.d`, `/var/lib/kubelet/plugins`, containerd socket)
- **Drill:** given a ServiceAccount that can't do something, diagnose and fix the RBAC in under 4 minutes.

### Week 8 — Troubleshooting boot camp (30% — the big one)
This is the week that decides your score. Work exclusively from the break-fix library in §4. Snapshot, break, restore, repeat.

- Node-level: `systemctl status kubelet`, `journalctl -u kubelet -f`, `crictl ps -a`, `crictl logs`
- Control plane: static pod manifests, `crictl` on the CP node when the API server itself is down
- Workload: `describe` events, `logs --previous`, `logs -c <container>`, exit codes, OOMKilled vs CrashLoopBackOff vs ImagePullBackOff
- Networking: endpoints, DNS from inside a pod, NetworkPolicy blocking
- Monitoring: `kubectl top nodes/pods`, container output streams
- **Build a personal triage checklist** and refine it all week. Mine would start: *is the node Ready? is the pod scheduled? is it running? does the service have endpoints? does DNS resolve?*

### Week 9 — Timed practice + Killer.sh session 1
- Take **Killer.sh attempt 1**. Expect a bad score. It is deliberately harder than the real exam — this is normal and widely reported.
- Work through every solution. The solutions page is arguably more valuable than the test itself.
- Re-run the same session as many times as you like within the 36-hour window. Do that.
- Do Killercoda CKA scenarios end-to-end, timed.

### Week 10 — Gaps, Killer.sh session 2, exam
- Attack your weakest domain from the Killer.sh feedback
- **Killer.sh attempt 2 (session B, different questions), 2–3 days before the exam**
- Two full timed runs on a single monitor with only the allowed docs open
- **Schedule the exam.** Don't wait for "ready" — the second attempt is included in the price.

---

## 4. The Break-Fix Library

Snapshot first. Have someone else run the break if you can — or write each one on a card, shuffle, and draw blind so you diagnose from symptoms rather than memory. **Target: identify the cause in under 3 minutes, fix in under 7.**

**Node & kubelet**
1. `systemctl stop kubelet` on a worker
2. Corrupt `/var/lib/kubelet/config.yaml` (break the YAML indentation)
3. Change `staticPodPath` in the kubelet config to a nonexistent directory
4. Re-enable swap on a node and restart the kubelet
5. Fill a node's disk with `fallocate` until DiskPressure evicts pods
6. Break the kubelet's client cert or its `/etc/kubernetes/kubelet.conf`

**Control plane**
7. Add a bogus flag to `/etc/kubernetes/manifests/kube-apiserver.yaml`
8. Point the API server at the wrong etcd endpoint
9. Change the API server's `--secure-port` and don't update anything else
10. Move `kube-scheduler.yaml` out of the manifests directory — then figure out *why* new pods are Pending
11. Corrupt `/etc/kubernetes/admin.conf`

**Networking**
12. Delete the CNI config from `/etc/cni/net.d/` → pods stuck ContainerCreating
13. Scale CoreDNS to 0 replicas
14. Break the CoreDNS Corefile ConfigMap
15. Give a Service a selector that matches nothing
16. Set `targetPort` to a port the container isn't listening on
17. Apply a default-deny NetworkPolicy and don't tell yourself

**Workloads & scheduling**
18. Typo an image name / reference a private registry with no imagePullSecret
19. Set memory limits below what the app needs → OOMKilled
20. Set CPU requests so high nothing can schedule
21. Taint every node with no matching toleration
22. Set a liveness probe that always fails
23. Bad ConfigMap key reference in `envFrom`

**Storage & RBAC**
24. PVC requesting a nonexistent StorageClass
25. PVC with an access mode the PV doesn't offer
26. Bind a Role in the wrong namespace and watch a ServiceAccount get denied
27. Grant a ClusterRole where a Role was needed (and vice versa)

**Lifecycle**
28. etcd backup → mass deletion → restore
29. Full kubeadm upgrade, v1.34 → v1.35
30. Break a worker's join, then re-join it with a fresh token

---

## 5. Speed Setup (practice with this from Week 1)

The exam pre-configures `k` and completion, but muscle memory for the rest is worth building. In your lab's `~/.bashrc`:

```bash
alias k=kubectl
export do='--dry-run=client -o yaml'
export now='--force --grace-period=0'
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```

`~/.vimrc`:

```vim
set expandtab
set tabstop=2
set shiftwidth=2
set number
set paste   " toggle off when typing normally
```

Commands worth burning into your fingers:

```bash
k run nginx --image=nginx $do > pod.yaml
k create deploy web --image=nginx --replicas=3 $do > deploy.yaml
k expose deploy web --port=80 --target-port=8080 --type=NodePort $do
k create cm app --from-literal=KEY=value $do
k create secret generic db --from-literal=pass=s3cr3t $do
k create role r1 --verb=get,list --resource=pods $do
k create rolebinding rb1 --role=r1 --serviceaccount=default:sa1 $do
k create job j1 --image=busybox $do -- /bin/sh -c "echo hi"
k get po -A -o wide --sort-by=.metadata.creationTimestamp
k get events -A --sort-by=.lastTimestamp
k explain deployment.spec.strategy --recursive
```

`kubectl explain --recursive` is often faster than searching the docs. Use it constantly in practice so you reach for it under pressure.

---

## 6. Resources

### Free — start here

| Resource | Link | Use it for |
|---|---|---|
| **CNCF official curriculum** | `github.com/cncf/curriculum` | The authoritative topic list. Print it. Tick items off. |
| **Kubernetes docs — Tasks section** | `kubernetes.io/docs/tasks/` | The exam's allowed docs. The Tasks walkthroughs *are* exam answers (etcd backup, kubeadm upgrade, etc.) |
| **kubectl cheat sheet** | `kubernetes.io/docs/reference/kubectl/cheatsheet/` | Allowed in the exam. Know where things are on this page. |
| **Killercoda — Killer Shell CKA** | `killercoda.com/killer-shell-cka` | Free browser scenarios from the same people who make the official simulator. Excellent. |
| **Killercoda playgrounds** | `killercoda.com` | Free throwaway clusters when you're away from the lab |
| **Gateway API docs** | `gateway-api.sigs.k8s.io` | Allowed in exam, new topic, learn the layout |
| **Helm docs** | `helm.sh/docs` | Allowed in exam |
| **techiescamp/cka-certification-guide** | `github.com/techiescamp/cka-certification-guide` | Community study notes kept current with v1.35 |
| **LFS158 Introduction to Kubernetes** | Linux Foundation / edX | Free foundational course |
| **TechWorld with Nana — Kubernetes course** | YouTube | Free, ~4 hrs, good conceptual grounding if you're new |
| **Kubernetes the Hard Way** | `github.com/kelseyhightower/kubernetes-the-hard-way` | Optional. Overkill for the exam, superb for genuinely understanding the control plane. A great Proxmox weekend project. |
| **KodeKloud community FAQ** | `github.com/kodekloudhub/community-faq` | Practical exam-logistics answers |

### Paid — worth it

| Resource | Approx. cost | Verdict |
|---|---|---|
| **KodeKloud CKA course (Mumshad Mannambeth)** | ~$15–20 on Udemy, or KodeKloud subscription | The consensus standard. Integrated browser labs and mock exams. If you buy one thing, buy this. Confirm the listing says it's updated for the post-2025 curriculum. |
| **Killer.sh simulator** | Included (2 sessions) | Non-negotiable. Harder than the real exam by design. Extra sessions purchasable if you want more. |
| **Benjamin Muschko, *CKA Study Guide*** (O'Reilly) | ~$40 | Well-structured book if you learn better by reading. Check the edition covers the revised curriculum. |
| **LFS258 Kubernetes Fundamentals** | $645 bundled with the exam | Official, thorough, but the KodeKloud + lab combo is better value for most people. |
| **THRIVE-ONE + exam bundle** | $625 | Worth it only if you'll take other LF certs (CKAD/CKS) within the year. |

> Killer.sh advertises a **KILLER30** code for 30% off CKA registration. Discount codes come and go — check for current promos before you buy, and look at the CNCF/LF sites during CloudNativeCon season.

### Community
CNCF Slack (`#kubernetes-users`), r/kubernetes, KodeKloud community forums. Recent "I passed" write-ups on dev.to are useful for current exam texture — just weight anything written before Feb 2025 accordingly.

---

## 7. Exam-Day Tactics

**Before**
- Run the PSI system check well in advance, not the night before
- **One monitor only.** Dual monitors are not supported. Practice at 15"+ / 1080p.
- Clear desk, clear walls, no papers, well-lit room, private space. You'll pan the webcam around.
- Government-issued photo ID with the name matching your LF profile **exactly**
- Wired ethernet if possible. Disable firewalls/antivirus that might block the secure browser. You cannot take it from a VM.
- Check in 15 minutes early

**During**
1. **Read every task first** and triage. Each task shows its point weight. Do high-weight-high-confidence first.
2. **SSH to the correct host.** Every single time. `exit` back to base when done.
3. **Never spend more than ~7 minutes on one task.** Flag it, move on, come back.
4. Use the notepad to track flagged tasks and anything awaiting reconciliation (a joining node takes a few minutes — don't sit and watch it).
5. Generate YAML, don't write it: `$do` and edit.
6. `kubectl explain --recursive` beats doc-searching for field names.
7. Bookmark the etcd backup/restore and kubeadm upgrade doc pages during the exam — the commands are long and fiddly.
8. **Verify your work.** Re-check completed tasks if you have time. People routinely find a missed sub-task worth several points.
9. **66% is the bar.** Skipping one hard task you genuinely can't do is a valid strategy. Partial credit exists — do the parts you can.

**After**
Results arrive by email within about 24 hours. If you don't pass, the retake is already paid for — reschedule quickly while everything is fresh.

---

## 8. Readiness Checklist

Don't schedule until most of these are true:

- [ ] I've built a kubeadm cluster from scratch at least three times, the last time without notes
- [ ] I've upgraded a cluster from one minor version to the next
- [ ] I can back up and restore etcd without looking up more than the cert paths
- [ ] I can diagnose a NotReady node in under 3 minutes
- [ ] I can diagnose a Service with no endpoints in under 3 minutes
- [ ] I've written a default-deny NetworkPolicy and then selectively opened traffic
- [ ] I've installed something with Helm and rolled it back
- [ ] I've built a Kustomize base + overlay and applied it with `-k`
- [ ] I've installed a CRD/operator and created a custom resource
- [ ] I've created a Gateway and an HTTPRoute and routed real traffic through them
- [ ] I've set up dynamic provisioning with a StorageClass and watched a PVC bind
- [ ] I've configured an HPA and seen it scale under load
- [ ] I can navigate `kubernetes.io/docs` to any needed page in under 30 seconds
- [ ] I've scored **75%+ on Killer.sh** within the time limit
- [ ] I've done two full 2-hour timed runs on a single monitor

---

## 9. One-Page Domain Reference

**Cluster Architecture, Installation & Configuration (25%)**
RBAC (Role/ClusterRole/bindings, ServiceAccounts, `auth can-i`) · infrastructure prep (swap, modules, sysctl, containerd, cgroups) · `kubeadm init/join/token/upgrade` · lifecycle (drain, cordon, version skew, etcd backup/restore) · HA control plane (stacked vs external etcd, load balancer) · Helm · Kustomize · CNI/CSI/CRI · CRDs and operators

**Workloads & Scheduling (15%)**
Deployments, rolling updates, rollback · ConfigMaps & Secrets · HPA · self-healing (ReplicaSets, probes, restart policies, DaemonSets, StatefulSets) · requests/limits, QoS, LimitRange, ResourceQuota · nodeSelector, affinity/anti-affinity, taints/tolerations, topology spread

**Services & Networking (20%)**
Pod-to-pod connectivity & the flat network model · NetworkPolicy (ingress/egress, selectors, ipBlock, default-deny) · ClusterIP/NodePort/LoadBalancer/ExternalName/headless + Endpoints & EndpointSlices · **Gateway API** (GatewayClass, Gateway, HTTPRoute) · Ingress controllers & resources · CoreDNS and cluster DNS naming

**Storage (10%)**
StorageClasses & dynamic provisioning · volume types, access modes (RWO/ROX/RWX/RWOP), reclaim policies · PV/PVC lifecycle and binding

**Troubleshooting (30%)**
Cluster and node failures (`journalctl -u kubelet`, `systemctl`, `crictl`) · control plane components (static pod manifests, etcd) · resource monitoring (`kubectl top`, metrics-server) · container output streams (`logs`, `--previous`, `-c`) · service and network failures (endpoints, DNS, NetworkPolicy)

---

*Verify exam details at `training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/` before you register — the Kubernetes version tracks upstream releases and the curriculum is updated quarterly.*

