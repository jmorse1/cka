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

---

### Step 1 — Create the base VM

Run all of this on the **Proxmox host**, as root.

```bash
# Move to where Proxmox keeps downloadable images
cd /var/lib/vz/template/iso

# Download the Ubuntu 24.04 "Noble" cloud image. Cloud images are pre-built,
# minimal, and designed to be configured at first boot by cloud-init —
# no interactive installer to click through.
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Create an empty VM shell with ID 9000. No disk yet — we attach that next.
#   --cpu host    passes your real CPU features through. The default kvm64 is
#                 slower and occasionally trips container runtime checks.
qm create 9000 --name ubuntu-2404-k8s --memory 4096 --cores 2 \
  --net0 virtio,bridge=vmbr0 --cpu host

# Import the downloaded cloud image as a disk belonging to VM 9000.
qm importdisk 9000 noble-server-cloudimg-amd64.img local-lvm

# Attach that imported disk as scsi0 using the virtio-scsi controller
# (best performance and TRIM support under Linux guests).
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0

# Add a cloud-init drive. This is a small virtual CD-ROM that Proxmox
# generates on each boot, carrying the username, password, SSH keys and
# network config you set with 'qm set'.
qm set 9000 --ide2 local-lvm:cloudinit

# Boot from the disk we just attached.
qm set 9000 --boot c --bootdisk scsi0

# Give the VM a serial console and make it the primary display.
# Ubuntu cloud images expect this. NOTE: this means the default noVNC
# console will render a blank screen — see Step 2.
qm set 9000 --serial0 socket --vga serial0

# Enable the QEMU guest agent so Proxmox can cleanly shut the VM down —
# which matters for taking consistent snapshots later.
qm set 9000 --agent enabled=1

# Cloud images ship a tiny (~2GB) disk. Grow it to something usable.
qm resize 9000 scsi0 +25G
```

### Step 2 — Set login credentials, then boot it

**This is the step that has to happen before templating.** Ubuntu cloud images have **no password and no configured user** out of the box — the `ubuntu` account is only created and unlocked when cloud-init runs, and cloud-init only knows what to create because you told Proxmox here. Skip this and the console will never offer you a login prompt.

```bash
# Tell cloud-init which user to create and what password to set.
# This is temporary — we delete it before templating.
qm set 9000 --ciuser ubuntu --cipassword 'temp-password-here'

# Install your SSH public key so you can log in without a password later.
qm set 9000 --sshkeys ~/.ssh/id_rsa.pub

# Boot it. First boot runs cloud-init, which creates the user, applies the
# SSH key, expands the filesystem to fill the resized disk, and sets up networking.
qm start 9000
```

Give it **30–60 seconds** for cloud-init to finish before trying to log in.

**How to reach the console.** Because we set `--vga serial0`, the default noVNC console shows nothing. Use one of:

- **Web UI:** select the VM → **Console** dropdown at top right → **xterm.js**
- **Host shell:** `qm terminal 9000` — attaches to the serial console. Exit with `Ctrl+O`.
- **SSH:** `ssh ubuntu@<ip>` once it has an address. Easiest option.

If xterm.js sits blank, **press Enter once** — the serial console usually needs a keystroke before it paints the login prompt.

Log in as `ubuntu` with the password you set.

### Step 3 — Prepare the node (inside the VM)

Do this **inside VM 9000**, before templating. Everything you install here gets baked into the template, so you do it once instead of three times.

```bash
sudo -i   # become root for the rest of this section
```

**Disable swap.** The kubelet refuses to start with swap enabled (unless you explicitly opt in, which the exam does not).

```bash
# Turn swap off right now, for this boot.
swapoff -a

# Comment out any swap entry in fstab so it stays off across reboots.
sed -i '/ swap / s/^/#/' /etc/fstab

# Cloud images sometimes enable zram or a swapfile via a systemd unit
# that fstab doesn't cover. Check for stragglers and disable what you find.
systemctl list-units --type=swap --all
```

**Load kernel modules and set sysctls.** Kubernetes networking needs bridged traffic to be visible to iptables, and needs IP forwarding on.

```bash
# Register the two modules so they load automatically on every boot.
#   overlay      — the filesystem containerd uses for image layers
#   br_netfilter — makes bridged traffic traverse iptables rules
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Load them now too, so we don't have to reboot.
modprobe overlay && modprobe br_netfilter

# Persist the network sysctls Kubernetes requires.
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply all sysctl config files immediately.
sysctl --system
```

**Install and configure containerd** — the container runtime (the CRI implementation) that the kubelet talks to.

```bash
apt-get update && apt-get install -y containerd

# containerd ships with no config file; generate the full default one.
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

# Switch the cgroup driver to systemd. This MUST match what the kubelet uses
# (systemd is the kubelet default). A mismatch causes pods to fail in ways
# that are genuinely painful to diagnose — worth knowing for the exam.
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

systemctl restart containerd
```

**Install the Kubernetes tools — deliberately at v1.34, not v1.35.** That way your first real lab exercise is a genuine `kubeadm upgrade` to v1.35, which is a graded exam topic you'd otherwise never practice.

```bash
# Prerequisites for adding an apt repo over HTTPS with a signing key.
apt-get install -y apt-transport-https ca-certificates curl gpg

# Download the signing key for the v1.34 package repo and convert it to
# the binary format apt expects.
mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the repo, pinned to that key.
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
  | tee /etc/apt/sources.list.d/kubernetes.list

apt-get update && apt-get install -y kubelet kubeadm kubectl

# Pin the versions. Without this, a routine 'apt upgrade' will silently
# bump your cluster components and break things. kubeadm upgrades are
# supposed to be deliberate — this is real-world practice too.
apt-mark hold kubelet kubeadm kubectl
```

### Step 4 — Clean up, then convert to a template

Still inside the VM. This is the part that prevents the clone collisions.

```bash
# Wipe cloud-init's state so it runs fresh on every clone rather than
# assuming it has already configured this machine.
cloud-init clean --logs

# Delete the SSH host keys. If you skip this, every node shares an identity
# and you'll get host key fingerprint collisions when hopping between them.
# They regenerate automatically on next boot.
rm -f /etc/ssh/ssh_host_*

# Blank the machine-id. This is the one that actually duplicates across
# clones and it breaks 'kubeadm join'. Emptying it (not deleting it) tells
# systemd to generate a fresh unique ID on first boot.
truncate -s 0 /etc/machine-id
rm -f /var/lib/dbus/machine-id
ln -s /etc/machine-id /var/lib/dbus/machine-id

history -c
shutdown -h now
```

Back on the **Proxmox host**, once the VM has fully stopped:

```bash
# Remove the temporary password so clones don't inherit it.
# From here on you log in with your SSH key.
qm set 9000 --delete cipassword

# Convert to a template. This makes the disk read-only and permanently
# non-bootable — which is exactly why every change above had to come first.
qm template 9000
```

> **Correction worth knowing:** you may see advice about duplicate `product_uuid` breaking `kubeadm join`. Proxmox assigns each VM its own SMBIOS UUID automatically, so that half isn't an issue here. It is **`/etc/machine-id`** that genuinely duplicates, and only if the VM was booted before templating — which yours was. Hence the cleanup above.

### Step 5 — Clone the nodes

> **Plan your three CIDRs before you type anything.** A Kubernetes cluster uses three separate address ranges and **none of them may overlap**. The examples below assume a LAN of `192.168.4.0/24`; substitute your own.
>
> | Range | Example | Notes |
> |---|---|---|
> | **LAN / node IPs** | `192.168.4.0/24` | Your real network |
> | **Pod CIDR** | `10.244.0.0/16` | Set with `--pod-network-cidr` |
> | **Service CIDR** | `10.96.0.0/12` | kubeadm default; spans `10.96.0.0`–`10.111.255.255` |
>
> Calico's documented default pod CIDR is `192.168.0.0/16`, which covers `192.168.0.0`–`192.168.255.255` and therefore **swallows any home LAN in the 192.168 range**. The failure is insidious: `kubeadm init` succeeds, then pods get addresses colliding with real hosts and connectivity fails intermittently later. Use `10.244.0.0/16` (or `172.16.0.0/16`) instead.

```bash
# Full clone (independent disk, not a linked clone — you want these
# independent so snapshot rollbacks stay simple).
qm clone 9000 701 --name cka-cp1 --full

# Give it a static IP via cloud-init, plus your SSH key.
# Adjust the subnet and gateway to match your LAN.
# NOTE: VM ID (701) and IP last octet (.201) intentionally differ here —
# Proxmox doesn't care, but keep your own mapping consistent and written down.
qm set 701 --ipconfig0 ip=192.168.4.201/24,gw=192.168.4.1 \
  --ciuser ubuntu --sshkeys ~/.ssh/id_rsa.pub

# Control plane gets a bit more RAM than the workers.
qm set 701 --memory 4096 --cores 2

qm start 701

# Repeat for the workers:
#   qm clone 9000 702 --name cka-w1 --full
#   qm set 702 --ipconfig0 ip=192.168.4.202/24,gw=192.168.4.1 --ciuser ubuntu --sshkeys ~/.ssh/id_rsa.pub
#   qm set 702 --memory 2048 --cores 2 && qm start 702
#   ...and 703 / cka-w2 the same way.
```

**Verify uniqueness before going further.** On each node:

```bash
cat /etc/machine-id              # must differ across all three nodes
cat /sys/class/dmi/id/product_uuid   # must also differ (Proxmox handles this)
hostname                          # should match the VM name
```

If `/etc/machine-id` comes back **empty on the template**, that's correct — systemd regenerates it on each clone's first boot. Empty in the template, unique after boot, is exactly the state you want.

### Step 6 — Bootstrap the cluster

On `cka-cp1`:

Before you start, note that `/etc/kubernetes/` on a prepared-but-uninitialised node contains **only** a `manifests/` directory holding an empty `.kubelet-keep` file. That placeholder ships with the `kubelet` package to stop dpkg removing the directory. No `admin.conf`, no `pki/`, no static pod manifests — all of those are created *by* `kubeadm init`. The kubelet will also be crash-looping, because it has no config to fetch yet. Both are normal at this point.

```bash
# Initialise the control plane. Tee the output — the join command and any
# failure detail scroll past quickly and you'll want both.
#   --pod-network-cidr is the range pods draw from. It must not overlap your
#     LAN or the service CIDR. See the CIDR table in Step 5.
#   --apiserver-advertise-address pins the API server to this node's IP
#     rather than letting kubeadm guess on a multi-interface box.
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.4.201 \
  2>&1 | tee ~/kubeadm-init.log

# Set up kubectl for your regular user (kubeadm prints these too).
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

If init fails partway, clean up before retrying — a half-initialised node fails preflight on the second attempt:

```bash
sudo kubeadm reset -f        # tears down whatever partial state exists
sudo rm -rf /etc/cni/net.d/  # stale CNI config confuses the next install
rm -f $HOME/.kube/config     # stale credentials pointing at a dead cluster
```

Common preflight rejections: swap still on (`swapon --show` should print nothing), `br_netfilter` not loaded (`lsmod | grep br_netfilter`), containerd unreachable (`sudo crictl info` should return JSON), or port 6443 still bound from a failed run.

Now install the CNI. **Always pin to a release tag.** The `master` branch is Calico's development branch: its manifests can be mid-refactor and mismatched against released images, which produces failures that look like cluster problems but aren't.

```bash
# Nodes stay NotReady until a CNI is installed — a symptom worth recognising,
# since it shows up in troubleshooting tasks.
# Calico rather than Flannel: Flannel does not implement NetworkPolicy,
# and NetworkPolicy is on the exam.
# Check github.com/projectcalico/calico/releases for the current stable tag.
curl -sL https://raw.githubusercontent.com/projectcalico/calico/v3.31.7/manifests/calico.yaml -o calico.yaml

# Inspect before applying. If CALICO_IPV4POOL_CIDR is commented out, Calico
# inherits the cluster's pod CIDR (what you passed to kubeadm) — which is what
# you want. If it's set to 192.168.0.0/16, change it to 10.244.0.0/16.
grep -A2 CALICO_IPV4POOL_CIDR calico.yaml

kubectl apply -f calico.yaml
kubectl -n kube-system rollout status deployment/calico-kube-controllers
```

Verify Calico actually used the range you intended, rather than assuming:

```bash
kubectl get ippools -o jsonpath='{.items[*].spec.cidr}'  # expect 10.244.0.0/16
kubectl get pods -A -o wide                              # pod IPs in 10.244.x.x
```

On each worker, run the `kubeadm join` command that `kubeadm init` printed. If you lost it:

```bash
# Regenerate a valid join command on the control plane.
# Tokens expire after 24h — this is how you recover, and it's an exam-relevant trick.
sudo kubeadm token create --print-join-command
```

Confirm everything landed:

```bash
kubectl get nodes -o wide        # all three Ready
kubectl get pods -A              # all system pods Running
```

### Step 7 — Drive the cluster from WSL

Running `kubectl` from a WSL Ubuntu instance on your workstation is more comfortable than working in a Proxmox console, and WSL2 reaches your LAN outbound without any special configuration. Do this before installing the add-ons in Step 8, since you will run all of those from here. Read the split-workflow warning at the end too — it decides how you practice.

#### 7a. Copy the kubeconfig

```bash
# scp on admin.conf FAILS — it's mode 0600 owned by root, so the ubuntu user
# can't read it. Pipe it through sudo on the remote side instead.
mkdir -p ~/.kube
ssh ubuntu@192.168.4.201 'sudo cat /etc/kubernetes/admin.conf' > ~/.kube/config
chmod 600 ~/.kube/config

kubectl get nodes
```

The `server:` field inside already points at `https://192.168.4.201:6443`, which is reachable from WSL2. Nothing here needs inbound access to WSL.

**If you already have other clusters in `~/.kube/config`**, merge rather than overwrite:

```bash
ssh ubuntu@192.168.4.201 'sudo cat /etc/kubernetes/admin.conf' > ~/.kube/cka.conf

# --flatten inlines the certs so the merged file stands alone.
KUBECONFIG=~/.kube/config:~/.kube/cka.conf kubectl config view --flatten > ~/.kube/merged
mv ~/.kube/merged ~/.kube/config && chmod 600 ~/.kube/config
```

```bash
# kubeadm's default context name is unhelpful once you have more than one.
kubectl config rename-context kubernetes-admin@kubernetes cka-lab
kubectl config use-context cka-lab
```

#### 7b. Match kubectl to the cluster version

Version skew beyond ±1 minor is unsupported. Your cluster is on v1.34 until you run the upgrade drill in Week 6, so install v1.34 here too.

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y kubectl

# Add the aliases and completion from section 5 to ~/.bashrc while you're here.
```

Install Helm in WSL too, not on the node — Steps 8a–8h are pure API calls and all work from here.

#### 7c. WSL gotchas

1. **Clock skew after the laptop sleeps.** WSL2's clock drifts from the Windows host on resume, and you get `x509: certificate has expired or is not yet valid` against a perfectly healthy cluster. The error points at certificates, so people burn real time on it. Fix: `sudo hwclock -s`.
2. **Cluster rebuild invalidates your kubeconfig.** Any `kubeadm reset` + `init` generates a new CA, giving `x509: certificate signed by unknown authority`. Re-copy `admin.conf`. A *snapshot rollback* is fine — certs are restored with everything else.
3. **`~/.kube/config` on the Windows filesystem.** Keep it in the WSL filesystem (`~`), not `/mnt/c/...`. The DrvFs mount can't represent Unix permissions, and kubectl warns about a world-readable config.

#### 7d. The split-workflow warning

**Roughly half the Troubleshooting domain — 30% of the exam — cannot be done from WSL at all.** Anything that touches the node itself needs a shell on the node:

| Works from WSL | Requires SSH to the node |
|---|---|
| All resource CRUD, `describe`, `logs`, `exec` | `journalctl -u kubelet`, `systemctl status kubelet` |
| Helm, Kustomize, all of Step 8 | `crictl ps` / `crictl logs` |
| RBAC, `auth can-i` | Editing `/etc/kubernetes/manifests/` |
| Port-forward, `top` | `etcdctl` backup and restore |
| Everything in Weeks 2–5 | Every `kubeadm` command |
| | Most of the break-fix library in §4 |

This is not a limitation to work around — **it mirrors the exam**. The CKA is SSH-based: you start on a `base` host that has no tools installed, and each task tells you which node to `ssh` into. Points are routinely lost by running a command on the wrong host.

So use WSL as your comfortable surface for resource work, but deliberately drill the `ssh cka-cp1` → work → `exit` cycle for anything node-level. Make it automatic:

```bash
# ~/.ssh/config in WSL — short names that match the exam's style
Host cka-cp1
    HostName 192.168.4.201
    User ubuntu
Host cka-w1
    HostName 192.168.4.202
    User ubuntu
Host cka-w2
    HostName 192.168.4.203
    User ubuntu
```

Now `ssh cka-cp1` works the way `ssh node01` will on exam day. Note that **nested SSH is not supported in the exam** — always `exit` back before hopping elsewhere.

### Step 8 — Install the add-ons

Without these, several curriculum domains are untestable in your lab. Install them in this order — MetalLB before ingress-nginx, and Gateway API CRDs before any gateway controller — because each depends on the one above it.

| Add-on | Unlocks | Domain |
|---|---|---|
| **Helm** | Chart install/upgrade/rollback | Cluster Architecture (25%) |
| **metrics-server** | `kubectl top`, HPA | Troubleshooting (30%), Workloads (15%) |
| **local-path-provisioner** | StorageClasses, dynamic provisioning | Storage (10%) |
| **MetalLB** | `type: LoadBalancer` gets a real IP | Services & Networking (20%) |
| **ingress-nginx** | Ingress resources | Services & Networking (20%) |
| **Gateway API + NGINX Gateway Fabric** | GatewayClass, Gateway, HTTPRoute | Services & Networking (20%) |
| **NFS server** *(optional)* | ReadWriteMany, reclaim policies | Storage (10%) |

Run all of this **from WSL** (Step 7) — every command here is a plain API call, so no SSH needed. **Join your workers first**: several of these need somewhere to schedule, and the control plane carries a `NoSchedule` taint by default.

#### 8a. Helm

```bash
# Official install script. Helm is a single static binary — no cluster-side
# component (Tiller was removed in Helm 3).
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

#### 8b. metrics-server

Nothing that consumes metrics works without this — `kubectl top` returns an error and every HPA sits at `<unknown>` targets.

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update

# --kubelet-insecure-tls is REQUIRED on a kubeadm cluster and is the single
# most common reason metrics-server fails. The kubelet's serving certificate
# is self-signed and not issued by the cluster CA, so metrics-server refuses
# the connection until you tell it to skip verification. Fine in a lab;
# in production you would fix the certs instead.
helm install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --set args="{--kubelet-insecure-tls}"

# Metrics take 30-60s to populate. An empty result immediately after install
# is normal, not a failure.
sleep 60 && kubectl top nodes
```

#### 8c. local-path-provisioner

Gives you a working StorageClass so PVCs bind dynamically instead of sitting `Pending` forever.

```bash
# Check github.com/rancher/local-path-provisioner/releases for the current
# tag and substitute it below.
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.31/deploy/local-path-storage.yaml

kubectl get storageclass
```

**Deliberately leave it non-default at first.** Create a PVC with no `storageClassName` and watch it hang in `Pending` — that exact symptom is a recurring exam scenario. Once you've seen it, make it default:

```bash
# The is-default-class annotation is what lets a PVC omit storageClassName.
kubectl patch storageclass local-path -p \
  '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### 8d. MetalLB

On bare metal there's no cloud controller, so `type: LoadBalancer` services stay `<pending>` forever. MetalLB hands out real LAN addresses.

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update
helm install metallb metallb/metallb --namespace metallb-system --create-namespace

kubectl -n metallb-system rollout status deployment/metallb-controller
```

The pool must be **real, routable addresses on your LAN that are outside your router's DHCP range** — otherwise your router will hand the same IPs to other devices and you'll chase intermittent conflicts.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lab-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.4.240-192.168.4.250
---
# L2Advertisement makes MetalLB answer ARP for those IPs. Without it the
# addresses are assigned but unreachable — a good "service has an IP but
# nothing connects" puzzle to have seen once.
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: lab-l2
  namespace: metallb-system
spec:
  ipAddressPools:
  - lab-pool
EOF
```

#### 8e. ingress-nginx

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# type=LoadBalancer makes the controller claim an address from MetalLB.
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer

# EXTERNAL-IP should show a MetalLB address, not <pending>.
kubectl -n ingress-nginx get svc
```

Note the IngressClass name it registers (`nginx`) — that's what goes in `spec.ingressClassName` on your Ingress resources. Omitting it when no default IngressClass exists means the Ingress is silently ignored, which is worth experiencing once.

#### 8f. Gateway API + a controller

CRDs first, always. Controllers crash on startup if the CRDs they watch don't exist.

```bash
# Standard channel = GA/beta resources: GatewayClass, Gateway, HTTPRoute,
# GRPCRoute, ReferenceGrant. That covers the exam; skip the experimental
# channel. --server-side avoids the annotation size limit these large CRDs hit.
kubectl apply --server-side -f \
  https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml

kubectl get crd | grep gateway
```

Then a controller that implements them. NGINX Gateway Fabric is the easiest on bare metal:

```bash
helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --create-namespace -n nginx-gateway \
  --set service.type=LoadBalancer

# A GatewayClass should now exist and report Accepted=True.
kubectl get gatewayclass
```

If `GatewayClass` shows `Accepted=False`, the controller isn't running or the CRD version is ahead of what it supports — check `kubectl -n nginx-gateway logs deploy/ngf-nginx-gateway-fabric`.

#### 8g. NFS server (optional, for ReadWriteMany)

`local-path` is ReadWriteOnce only. If you want to practice RWX and reclaim policies against a real backend, stand up a small VM:

```bash
# On the NFS VM
sudo apt-get install -y nfs-kernel-server
sudo mkdir -p /srv/nfs/k8s && sudo chown nobody:nogroup /srv/nfs/k8s
echo '/srv/nfs/k8s 192.168.4.0/24(rw,sync,no_subtree_check,no_root_squash)' \
  | sudo tee -a /etc/exports
sudo exportfs -ra

# On every Kubernetes node — the client package must be present or mounts
# fail with a confusing "wrong fs type" error.
sudo apt-get install -y nfs-common
```

Then install the dynamic provisioner:

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm install nfs-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=192.168.4.210 \
  --set nfs.path=/srv/nfs/k8s
```

#### 8h. Verify, then snapshot

```bash
kubectl get nodes                      # all Ready
kubectl get pods -A                    # everything Running
kubectl top nodes                      # metrics-server alive
kubectl get storageclass               # local-path present
kubectl get gatewayclass               # Accepted=True
kubectl -n ingress-nginx get svc       # EXTERNAL-IP assigned
helm list -A                           # every release deployed
```

When all of that is green, **take a fresh snapshot of all three VMs** (see the snapshot workflow below) and name it something like `clean-v134-addons`. That becomes your restore point for the entire break-fix library.

### Proxmox gotchas that will waste your evening

1. **Blank console.** Expected — `--vga serial0` means you need xterm.js or `qm terminal`, not noVNC. Press Enter to wake the prompt.
2. **No login prompt / password rejected.** You didn't set `--ciuser`/`--cipassword` before first boot, or cloud-init hasn't finished. Set them, then `qm stop` and `qm start` (a reboot alone may not re-run cloud-init).
3. **Duplicate `machine-id`.** Breaks `kubeadm join`. Fixed by Step 4.
4. **Calico IP autodetection picks the wrong NIC.** Calico defaults to `first-found`, which can latch onto a virtual or secondary interface. The `calico-node` log names the address it detected. Pin it: `kubectl -n kube-system set env daemonset/calico-node IP_AUTODETECTION_METHOD=can-reach=192.168.4.1` (or `interface=ens18`).
5. **`calico-kube-controllers` in CrashLoopBackOff with `mkdir /status: permission denied`.** A manifest packaging fault, almost always from using the `master` branch. It writes health state to `/status` and its probes read it back; if the directory can't be created the probes fail and the kubelet kills a container that is otherwise working fine. Reinstall from a pinned tag. To unblock immediately: `kubectl -n kube-system patch deployment calico-kube-controllers -p '{"spec":{"template":{"spec":{"volumes":[{"name":"status","emptyDir":{}}],"containers":[{"name":"calico-kube-controllers","volumeMounts":[{"name":"status","mountPath":"/status"}]}]}}}}'`
6. **Memory ballooning.** Disable it, or set minimum equal to maximum. The kubelet becomes very unhappy when RAM is pulled out from under it, and you'll waste hours chasing a phantom problem.
7. **CPU type.** Use `host`. The default `kvm64` is slower and can trip runtime checks.
8. **Swap creeping back.** Re-check after any template rebuild.

### The snapshot workflow — your secret weapon

Once the cluster is healthy with add-ons installed:

```bash
# Shut down ALL nodes first. A snapshot of a running multi-node cluster
# captures etcd mid-write and gives you an inconsistent restore.
for id in 701 702 703; do qm shutdown $id; done

# Wait until all three show 'stopped', then snapshot each one.
for id in 701 702 703; do
  qm snapshot $id clean-v134 --description "healthy cluster, pre-break"
done

for id in 701 702 703; do qm start $id; done
```

To restore after you've wrecked something:

```bash
# Roll all three back together to the same point in time.
for id in 701 702 703; do
  qm stop $id
  qm rollback $id clean-v134
  qm start $id
done
```

**Always snapshot and restore all nodes together.** Restoring a single node into a cluster whose etcd has moved on gives you certificate and clock-skew errors that teach you nothing useful. After any rollback, give the cluster 2–3 minutes and check `kubectl get nodes` before concluding something is broken.

Take a fresh snapshot at each milestone: `clean-v134`, `clean-v135-upgraded`, `with-gateway-api`, and so on.

---

## 3. The 10-Week Plan

Assumes **8–10 hours/week**. Compress to 6 weeks at 15+ hrs/week if you already run Kubernetes at work; stretch to 14 if you're new to containers.

**The 70/30 rule: at least 70% of every session is hands-on in a terminal.** Watching videos feels like progress and isn't. If you finish a study session without having typed `kubectl` fifty times, that session didn't count.

### Week 0 — Lab build
Everything in Phase 0, Steps 1-8. End state: 3-node cluster on v1.34, Calico pinned to a release tag, all add-ons installed and verified, `clean-v134-addons` snapshot taken, kubectl working from WSL with SSH host aliases configured. Also set up your `.bashrc` and `.vimrc` (see §5).

### Week 1 — Architecture & kubectl fluency
- Control plane components: what api-server, scheduler, controller-manager, etcd, kubelet, kube-proxy each actually do
- Static pods in `/etc/kubernetes/manifests` — how they differ from normal pods, and that you never `apply` them
- `kubectl` verbs, `-o wide`, `-o yaml`, `-o jsonpath`, `--dry-run=client -o yaml`, `explain`, `describe`, `api-resources`
- Contexts and namespaces: `kubectl config use-context`, `set-context --current --namespace=x`
- **Drill:** recreate every core resource from imperative commands only. No copy-paste from docs.

### Week 2 — Workloads & Scheduling (15%)
- Deployments: rolling update strategy, `maxSurge`/`maxUnavailable`, `rollout status/history/undo`
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
- Reclaim policies: Retain, Delete — observe the actual behavior on PV deletion
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
- **A CrashLoopBackOff does not mean the process is failing.** A container whose logs look healthy can still be killed because its *probe* can't confirm it. When logs and errors disagree, `kubectl describe pod` and read the probe definitions and `Liveness probe failed` events. Distinguishing "the app is broken" from "the probe can't verify the app" is exam-grade skill.
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
# Short alias — you'll type this hundreds of times in two hours.
alias k=kubectl

# Generate a manifest without contacting the cluster, so you can edit
# rather than write YAML from scratch. Usage: k run nginx --image=nginx $do
export do='--dry-run=client -o yaml'

# Delete immediately instead of waiting out the 30s grace period.
export now='--force --grace-period=0'

# Tab completion for resource names, namespaces, and flags.
source <(kubectl completion bash)

# Make completion work through the 'k' alias too — it doesn't by default.
complete -o default -F __start_kubectl k
```

`~/.vimrc`:

```vim
set expandtab      " spaces, never tabs — YAML rejects tabs outright
set tabstop=2      " a tab reads as 2 columns
set shiftwidth=2   " indent/outdent by 2, the YAML convention
set number         " line numbers, so error messages mean something
```

Commands worth burning into your fingers:

```bash
# Scaffold a single pod manifest.
k run nginx --image=nginx $do > pod.yaml

# Scaffold a 3-replica deployment.
k create deploy web --image=nginx --replicas=3 $do > deploy.yaml

# Scaffold a service in front of an existing deployment.
k expose deploy web --port=80 --target-port=8080 --type=NodePort $do

# ConfigMap and Secret from literal values.
k create cm app --from-literal=KEY=value $do
k create secret generic db --from-literal=pass=s3cr3t $do

# RBAC: a namespaced role, then bind it to a ServiceAccount.
k create role r1 --verb=get,list --resource=pods $do
k create rolebinding rb1 --role=r1 --serviceaccount=default:sa1 $do

# A one-shot Job.
k create job j1 --image=busybox $do -- /bin/sh -c "echo hi"

# Every pod in the cluster, with node placement, newest last.
k get po -A -o wide --sort-by=.metadata.creationTimestamp

# Cluster-wide event feed, most recent last — the fastest first move
# on almost any "why is this broken" question.
k get events -A --sort-by=.lastTimestamp

# Look up field names without leaving the terminal. Often faster than
# searching the docs, and it works when you can't recall the exact path.
k explain deployment.spec.strategy --recursive
```

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
