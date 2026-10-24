# CKA Lab Workbook — Weeks 1–10

*Companion to the CKA Study Guide. That document covers exam facts, the lab build, and resources. This one is the actual work: numbered exercises, per week, with commands and verification.*

---

## How to use this

Each week has **Exercises** (build something, observe what happens), **Drills** (break something, fix it under time pressure), and a **Done when** checklist. Don't move on until the checklist passes.

**Run everything from WSL** unless an exercise says otherwise. Node-level work is marked 🔴 **on-node** — SSH to the node, do the work, `exit`. Build that habit now; the exam is SSH-based.

**Snapshot discipline:** roll back to `clean-v134-addons` between drill sessions. Every drill assumes a healthy starting cluster.

### Calibration for your background

You've run workloads on AKS, so Weeks 2–3 will feel familiar — move fast, but don't skip the imperative-generation practice, because speed is what the exam actually grades. The material AKS never exposed you to is where the marks are:

| Spend less time | Spend more time |
|---|---|
| Deployments, ReplicaSets, probes | Static pods, `/etc/kubernetes/manifests/` |
| ConfigMaps, Secrets, env injection | kubeadm init/join/upgrade |
| Services, basic DNS | etcd backup and restore |
| Prometheus-style monitoring concepts | kubelet failure modes, `crictl` |
| Container/Docker fundamentals | Certificate paths and expiry |

Weeks 6 and 8 are the ones that decide your result. Budget extra time there and take it from Weeks 2–3 if you need to.

---

## Week 1 — Architecture & kubectl fluency

**Goal:** know what every control plane component does, and generate any resource from memory in seconds.

### Exercise 1.1 — Map the control plane 🔴 on-node

```bash
ssh cka-cp1
```

```bash
# The four control plane components run as STATIC PODS — the kubelet reads
# these files directly from disk. There is no Deployment, no ReplicaSet,
# and the scheduler is not involved in placing them.
sudo ls -l /etc/kubernetes/manifests/

# Read the API server manifest end to end. Note every flag: --etcd-servers,
# --service-cluster-ip-range, --client-ca-file, --secure-port.
# Exam tasks frequently involve editing exactly this file.
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

Now prove the static-pod relationship to yourself:

```bash
# Move the scheduler manifest out of the directory.
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/

# Within ~20 seconds the pod disappears. The kubelet noticed the file vanish.
kubectl -n kube-system get pods | grep scheduler

# Create a pod — it stays Pending forever. Nothing is assigning it a node.
kubectl run test --image=nginx
kubectl get pod test -o wide        # NODE column empty
kubectl describe pod test           # no scheduler events at all

# Put it back. The pod returns on its own.
sudo mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
kubectl get pod test -o wide        # now scheduled
kubectl delete pod test
```

**Why this matters:** "pods stuck Pending with no events" is a classic exam symptom, and the cause is a control plane component rather than anything about the pod.

```bash
exit
```

### Exercise 1.2 — Output formats and jsonpath

```bash
# -o wide adds node placement and pod IPs. Use it by default.
kubectl get pods -A -o wide

# Extract single values. Worth practicing because exam tasks often say
# "write the result to /opt/answer.txt".
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'

kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Custom columns are often more readable than jsonpath for tabular answers.
kubectl get pods -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,NODE:.spec.nodeName

# Sorting — useful for "find the most recently created X" tasks.
kubectl get pods -A --sort-by=.metadata.creationTimestamp
kubectl get events -A --sort-by=.lastTimestamp
```

### Exercise 1.3 — Imperative generation (do this until it's reflexive)

Set up the aliases from §5 of the guide first, then generate each of these **without looking anything up**:

```bash
k run nginx --image=nginx $do > 1.yaml
k create deploy web --image=nginx --replicas=3 $do > 2.yaml
k expose deploy web --port=80 --target-port=8080 $do > 3.yaml
k create cm settings --from-literal=LOG_LEVEL=debug $do > 4.yaml
k create secret generic dbcreds --from-literal=password=hunter2 $do > 5.yaml
k create sa builder $do > 6.yaml
k create job onetime --image=busybox $do -- /bin/sh -c "echo done" > 7.yaml
k create cronjob nightly --image=busybox --schedule="0 2 * * *" $do -- /bin/sh -c "echo run" > 8.yaml
k create ns staging $do > 9.yaml
k create quota compute --hard=cpu=2,memory=4Gi $do > 10.yaml
```

Read each generated file. Knowing the shape of the YAML is what lets you edit fast.

**Target: all ten in under four minutes.** Repeat daily this week.

### Exercise 1.4 — Contexts and namespaces

```bash
kubectl config get-contexts
kubectl config current-context

# Pin a default namespace so you stop typing -n everywhere.
# Exam tasks specify a namespace constantly — missing it fails the task.
kubectl config set-context --current --namespace=kube-system
kubectl get pods                     # now defaults to kube-system
kubectl config set-context --current --namespace=default
```

### Exercise 1.5 — kubectl explain

```bash
# Faster than searching the docs, and works when you can't recall the path.
kubectl explain pod.spec.containers.resources
kubectl explain deployment.spec.strategy --recursive
kubectl explain networkpolicy.spec --recursive | head -40
```

### Drills

Snapshot first. Library items **10** and **1**:

- Move `kube-scheduler.yaml` out of `manifests/` (blind — have it done, then diagnose)
- `sudo systemctl stop kubelet` on `cka-w1`, then diagnose from `kubectl get nodes` alone

### Done when

- [ ] I can name every file in `/etc/kubernetes/manifests/` and what it does
- [ ] I can explain why static pods don't need the scheduler
- [ ] All ten imperative generations in under 4 minutes, no lookups
- [ ] I can extract a node's InternalIP with jsonpath without help
- [ ] I diagnosed a stopped kubelet in under 3 minutes

---

## Week 2 — Workloads & Scheduling (15%)

**Goal:** speed. You know this material from AKS — the exam grades how fast you can produce it.

### Exercise 2.1 — Rolling updates and rollback

```bash
k create deploy web --image=nginx:1.24 --replicas=4
kubectl rollout status deploy/web

# Watch the surge/unavailable behavior in a second terminal:
#   kubectl get pods -w
kubectl set image deploy/web nginx=nginx:1.25
kubectl rollout status deploy/web

kubectl rollout history deploy/web

# Break it deliberately with an image that doesn't exist.
kubectl set image deploy/web nginx=nginx:doesnotexist
kubectl get pods            # some ImagePullBackOff, old ones still serving
kubectl rollout status deploy/web    # hangs — this is the symptom

kubectl rollout undo deploy/web
kubectl rollout status deploy/web

# Tune the strategy and observe the difference.
kubectl patch deploy web -p \
  '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":0,"maxUnavailable":1}}}}'
```

**Observe:** with `maxSurge: 0` the rollout is slower and capacity dips. That trade-off is what the fields mean.

### Exercise 2.2 — ConfigMaps and Secrets

```bash
k create cm appcfg --from-literal=LOG_LEVEL=debug --from-literal=REGION=us-east
k create secret generic dbcreds --from-literal=user=admin --from-literal=password=hunter2

# Secrets are base64, NOT encrypted. Prove it:
kubectl get secret dbcreds -o jsonpath='{.data.password}' | base64 -d; echo
```

Mount both three ways — as individual env vars, as `envFrom`, and as a volume:

```bash
cat <<'YAMLEOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cfgdemo
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    env:
    - name: SINGLE_VALUE
      valueFrom:
        configMapKeyRef: {name: appcfg, key: LOG_LEVEL}
    envFrom:
    - configMapRef: {name: appcfg}
    - secretRef: {name: dbcreds}
    volumeMounts:
    - name: cfg
      mountPath: /etc/config
  volumes:
  - name: cfg
    configMap: {name: appcfg}
YAMLEOF

kubectl exec cfgdemo -- env | sort
kubectl exec cfgdemo -- ls /etc/config
kubectl exec cfgdemo -- cat /etc/config/LOG_LEVEL
```

**Key behavior to internalize:** edit the ConfigMap and re-check. The *volume* updates within a minute or so; the *env vars* do not until the pod restarts. That asymmetry is exam material.

### Exercise 2.3 — Probes

```bash
cat <<'YAMLEOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: probedemo
spec:
  containers:
  - name: web
    image: nginx
    readinessProbe:
      httpGet: {path: /, port: 80}
      initialDelaySeconds: 3
    livenessProbe:
      httpGet: {path: /nonexistent, port: 80}   # deliberately fails
      periodSeconds: 5
      failureThreshold: 3
YAMLEOF

# Watch the restart counter climb. The container is fine — the PROBE fails.
kubectl get pod probedemo -w
kubectl describe pod probedemo | tail -20
```

**This is the lesson from your Calico crashloop, generalized:** CrashLoopBackOff often means the probe can't verify a healthy process. Always read the probe definition before assuming the app is broken.

### Exercise 2.4 — Requests, limits, QoS

```bash
# Three pods, three QoS classes.
kubectl run guaranteed --image=nginx \
  --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx","resources":{"requests":{"cpu":"100m","memory":"128Mi"},"limits":{"cpu":"100m","memory":"128Mi"}}}]}}'

kubectl run burstable --image=nginx \
  --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx","resources":{"requests":{"cpu":"100m","memory":"64Mi"},"limits":{"memory":"128Mi"}}}]}}'

kubectl run besteffort --image=nginx

kubectl get pod guaranteed burstable besteffort \
  -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
```

Requests equal to limits gives Guaranteed; no resources at all gives BestEffort. **BestEffort is evicted first under node pressure** — which is why drill 5 (disk pressure) behaves the way it does.

### Exercise 2.5 — Scheduling controls

```bash
kubectl label node cka-w1 disk=ssd
kubectl run fast --image=nginx --overrides='{"spec":{"nodeSelector":{"disk":"ssd"}}}'
kubectl get pod fast -o wide        # lands on cka-w1

# Taints
kubectl taint node cka-w2 dedicated=db:NoSchedule
kubectl run nowhere --image=nginx --overrides='{"spec":{"nodeSelector":{"kubernetes.io/hostname":"cka-w2"}}}'
kubectl get pod nowhere             # Pending
kubectl describe pod nowhere | grep -A3 Events    # "had taint ... that the pod didn't tolerate"

# Tolerate it
kubectl delete pod nowhere
cat <<'YAMLEOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: {name: tolerant}
spec:
  nodeSelector: {kubernetes.io/hostname: cka-w2}
  tolerations:
  - key: dedicated
    operator: Equal
    value: db
    effect: NoSchedule
  containers: [{name: nginx, image: nginx}]
YAMLEOF

kubectl get pod tolerant -o wide
kubectl taint node cka-w2 dedicated=db:NoSchedule-   # trailing dash removes
```

### Exercise 2.6 — HPA

```bash
kubectl create deploy php --image=registry.k8s.io/hpa-example
kubectl set resources deploy php --requests=cpu=200m    # HPA NEEDS requests
kubectl expose deploy php --port=80

kubectl autoscale deploy php --cpu-percent=50 --min=1 --max=10
kubectl get hpa php -w

# Generate load in another terminal
kubectl run loader --rm -it --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://php; done"
```

**If the HPA shows `<unknown>`**, the cause is one of two things: metrics-server isn't working, or the deployment has no CPU *requests*. Percentage targets are computed against requests — with no request there's nothing to take a percentage of.

### Drills

Library items **18, 19, 20, 21, 22, 23**. Do them blind.

### Done when

- [ ] I can do a rolling update, break it, and roll back without notes
- [ ] I can explain why env vars don't update but volume-mounted ConfigMaps do
- [ ] I can name all three QoS classes and which is evicted first
- [ ] I can add and remove a taint and write a matching toleration from memory
- [ ] I can fix an HPA showing `<unknown>`

---

## Week 3 — Services, Endpoints, CoreDNS

**Goal:** diagnose a broken Service in under three minutes. This is the highest-frequency troubleshooting scenario on the exam.

### Exercise 3.1 — Service types

```bash
k create deploy web --image=nginx --replicas=3
kubectl expose deploy web --port=80 --name=web-clusterip
kubectl expose deploy web --port=80 --type=NodePort --name=web-nodeport
kubectl expose deploy web --port=80 --type=LoadBalancer --name=web-lb

kubectl get svc
```

`web-lb` gets a MetalLB address from your pool. On a cluster without MetalLB it would sit `<pending>` forever — worth remembering as a symptom.

```bash
# Reach each one
kubectl run tester --rm -it --image=busybox --restart=Never -- wget -qO- http://web-clusterip
curl http://192.168.4.202:$(kubectl get svc web-nodeport -o jsonpath='{.spec.ports[0].nodePort}')
curl http://$(kubectl get svc web-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

### Exercise 3.2 — Endpoints: the debugging tool that matters

```bash
# A Service is a selector plus a port mapping. The selector produces endpoints.
# No endpoints = no backends = connection refused, regardless of how
# correct the Service YAML looks.
kubectl get endpoints web-clusterip
kubectl get endpointslices -l kubernetes.io/service-name=web-clusterip
```

Now break it three ways and watch what each looks like:

```bash
# 1. Wrong selector — endpoints go empty
kubectl patch svc web-clusterip -p '{"spec":{"selector":{"app":"nonexistent"}}}'
kubectl get endpoints web-clusterip          # <none>
kubectl patch svc web-clusterip -p '{"spec":{"selector":{"app":"web"}}}'

# 2. Wrong targetPort — endpoints EXIST but connections fail.
#    This is the nastier one: the obvious check looks fine.
kubectl patch svc web-clusterip -p '{"spec":{"ports":[{"port":80,"targetPort":9999}]}}'
kubectl get endpoints web-clusterip          # populated!
kubectl run t --rm -it --image=busybox --restart=Never -- wget -qO- --timeout=3 http://web-clusterip
kubectl patch svc web-clusterip -p '{"spec":{"ports":[{"port":80,"targetPort":80}]}}'

# 3. No ready backends — probe failures remove pods from endpoints
kubectl scale deploy web --replicas=0
kubectl get endpoints web-clusterip
kubectl scale deploy web --replicas=3
```

**Build your triage order now:** Service exists → endpoints populated → targetPort matches container port → pods Ready → DNS resolves.

### Exercise 3.3 — CoreDNS

```bash
kubectl run dnstest --rm -it --image=busybox:1.28 --restart=Never -- sh
```

Inside:

```sh
nslookup web-clusterip
nslookup web-clusterip.default.svc.cluster.local
nslookup kubernetes.default
cat /etc/resolv.conf     # nameserver is the CoreDNS ClusterIP; note the search domains
exit
```

Use `busybox:1.28` specifically — later busybox images have a broken `nslookup` that produces confusing failures.

```bash
# Inspect the CoreDNS config. Exam tasks sometimes have you edit this.
kubectl -n kube-system get cm coredns -o yaml
kubectl -n kube-system get svc kube-dns
```

### Exercise 3.4 — Headless services and StatefulSets

```bash
kubectl create svc clusterip web-headless --clusterip="None" --tcp=80:80
kubectl patch svc web-headless -p '{"spec":{"selector":{"app":"web"}}}'

# A headless service returns pod IPs directly, not a single virtual IP.
kubectl run t --rm -it --image=busybox:1.28 --restart=Never -- nslookup web-headless
```

### Drills

Library items **13, 14, 15, 16**. Have them applied blind; diagnose from symptoms.

### Done when

- [ ] I check endpoints first, reflexively, on any service problem
- [ ] I can distinguish "no endpoints" from "endpoints but wrong targetPort" by symptom
- [ ] I can resolve a service by FQDN from inside a pod
- [ ] I can find and read the CoreDNS Corefile
- [ ] I diagnosed all four drills in under 3 minutes each

---

## Week 4 — Ingress, Gateway API, NetworkPolicy

**Goal:** the newest exam material. Gateway API especially — it was added in the 2025 revision and most older study material doesn't cover it.

### Exercise 4.1 — Ingress

```bash
k create deploy shop --image=nginx && kubectl expose deploy shop --port=80
k create deploy blog --image=httpd && kubectl expose deploy blog --port=80

cat <<'YAMLEOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: site
spec:
  ingressClassName: nginx          # omit this with no default class = silently ignored
  rules:
  - host: shop.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: {service: {name: shop, port: {number: 80}}}
  - host: blog.lab.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: {service: {name: blog, port: {number: 80}}}
YAMLEOF

kubectl get ingress
INGRESS_IP=$(kubectl -n ingress-nginx get svc ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Host-based routing — the Host header is what selects the rule.
curl -H "Host: shop.lab.local" http://$INGRESS_IP
curl -H "Host: blog.lab.local" http://$INGRESS_IP
```

Then delete `ingressClassName` and re-apply. Nothing routes, and there is **no error** — the Ingress is simply unclaimed. Recognizing that silence is the skill.

### Exercise 4.2 — Gateway API

The mental model that questions lean on: **GatewayClass** is the infrastructure type (set up by the provider), **Gateway** is an instance with listeners (cluster operator), **HTTPRoute** attaches to it and defines routing (app developer). That role separation is the design's whole purpose.

```bash
kubectl get gatewayclass

cat <<'YAMLEOF' | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: lab-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces: {from: Same}
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: shop-route
spec:
  parentRefs:
  - name: lab-gateway
  hostnames: ["shop.gw.local"]
  rules:
  - matches:
    - path: {type: PathPrefix, value: /}
    backendRefs:
    - name: shop
      port: 80
YAMLEOF

# Both must report Accepted=True. This is where you debug.
kubectl get gateway lab-gateway -o yaml | grep -A10 status:
kubectl describe httproute shop-route
```

```bash
GW_IP=$(kubectl get gateway lab-gateway -o jsonpath='{.status.addresses[0].value}')
curl -H "Host: shop.gw.local" http://$GW_IP
```

Then practice **traffic splitting**, which Ingress can't express natively:

```bash
kubectl patch httproute shop-route --type=json -p='[
  {"op":"replace","path":"/spec/rules/0/backendRefs","value":[
    {"name":"shop","port":80,"weight":90},
    {"name":"blog","port":80,"weight":10}]}]'
```

### Exercise 4.3 — NetworkPolicy

```bash
kubectl create ns secure
kubectl -n secure create deploy api --image=nginx && kubectl -n secure expose deploy api --port=80
kubectl -n secure create deploy client --image=busybox -- sleep 3600

# Baseline: works, because with no policies everything is allowed.
kubectl -n secure exec deploy/client -- wget -qO- --timeout=3 http://api
```

```bash
# Default deny ALL ingress in the namespace.
# An empty podSelector selects every pod; an empty policyTypes ingress list
# means "no ingress allowed".
cat <<'YAMLEOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: default-deny-ingress, namespace: secure}
spec:
  podSelector: {}
  policyTypes: [Ingress]
YAMLEOF

kubectl -n secure exec deploy/client -- wget -qO- --timeout=3 http://api   # times out
```

```bash
# Re-open exactly one path. Policies are ADDITIVE — this doesn't replace
# the deny, it adds an allowance on top of it.
cat <<'YAMLEOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: allow-client-to-api, namespace: secure}
spec:
  podSelector:
    matchLabels: {app: api}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels: {app: client}
    ports:
    - protocol: TCP
      port: 80
YAMLEOF

kubectl -n secure exec deploy/client -- wget -qO- --timeout=3 http://api   # works again
```

Now add an egress default-deny and watch **DNS break first** — a detail that catches people out, since name resolution is itself network traffic:

```bash
cat <<'YAMLEOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: default-deny-egress, namespace: secure}
spec:
  podSelector: {}
  policyTypes: [Egress]
YAMLEOF

kubectl -n secure exec deploy/client -- wget -qO- --timeout=3 http://api
# Fails at resolution, not connection. You must explicitly allow UDP/TCP 53
# to kube-dns before anything else works.
```

### Drills

Library item **17**, plus: apply a policy that allows the wrong port, and one that selects the wrong pods.

### Done when

- [ ] I can write an Ingress with host and path rules from memory
- [ ] I can explain GatewayClass vs Gateway vs HTTPRoute and who owns each
- [ ] I can check `Accepted` status on a Gateway and an HTTPRoute
- [ ] I can write a default-deny and a targeted allow without notes
- [ ] I know why egress policies break DNS and how to fix it

---

## Week 5 — Storage (10%)

**Goal:** small domain, quick wins. Focus on the `Pending` PVC symptoms.

### Exercise 5.1 — Static PV and PVC binding

```bash
cat <<'YAMLEOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata: {name: manual-pv}
spec:
  capacity: {storage: 1Gi}
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath: {path: /mnt/data}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: manual-pvc}
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: manual
  resources: {requests: {storage: 1Gi}}
YAMLEOF

kubectl get pv,pvc      # both should show Bound
```

Now produce each failure mode deliberately:

```bash
# A. storageClassName mismatch
kubectl delete pvc manual-pvc
kubectl create -f - <<'YAMLEOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: wrong-class}
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nonexistent
  resources: {requests: {storage: 1Gi}}
YAMLEOF
kubectl get pvc wrong-class                   # Pending
kubectl describe pvc wrong-class | tail -5    # the reason is here

# B. Requesting more than any PV offers
# C. Access mode the PV doesn't support
```

**Always `describe` a Pending PVC.** The events name the exact reason, every time.

### Exercise 5.2 — Dynamic provisioning

```bash
kubectl get storageclass

# With local-path as default, no storageClassName needed at all.
cat <<'YAMLEOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: dynamic-pvc}
spec:
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 1Gi}}
YAMLEOF

# NOTE: local-path uses WaitForFirstConsumer — the PVC stays Pending until
# a pod actually mounts it. That is CORRECT behavior, not a fault.
kubectl get pvc dynamic-pvc

kubectl run user --image=nginx --overrides='
{"spec":{"containers":[{"name":"nginx","image":"nginx","volumeMounts":[{"name":"d","mountPath":"/data"}]}],
"volumes":[{"name":"d","persistentVolumeClaim":{"claimName":"dynamic-pvc"}}]}}'

kubectl get pvc dynamic-pvc    # now Bound, PV auto-created
kubectl get pv
```

That `WaitForFirstConsumer` distinction matters: `volumeBindingMode` determines whether Pending is a bug or expected.

### Exercise 5.3 — Reclaim policies

```bash
# Predict the outcome before running each.
kubectl get pv -o custom-columns=NAME:.metadata.name,RECLAIM:.spec.persistentVolumeReclaimPolicy,STATUS:.status.phase

kubectl delete pod user
kubectl delete pvc dynamic-pvc
kubectl get pv     # Delete policy -> PV is gone

# Retain leaves the PV behind in Released state, NOT available for rebinding
# until an admin clears spec.claimRef.
kubectl patch pv manual-pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

### Drills

Library items **24, 25**.

### Done when

- [ ] I can diagnose a Pending PVC from `describe` output alone
- [ ] I can explain `WaitForFirstConsumer` vs `Immediate`
- [ ] I can predict what happens to a PV on Retain vs Delete
- [ ] I can create a StorageClass and make it default

---

## Week 6 — Cluster Lifecycle (part of the 25%)

**Goal:** the domain AKS completely hid from you. Take your time here — this and Week 8 are where the exam is won.

### Exercise 6.1 — etcd backup and restore 🔴 on-node

Near-certain to appear on the exam. Practice until only the cert paths need looking up.

```bash
ssh cka-cp1
sudo -i
```

```bash
# etcdctl may not be installed. Get it, or use the etcd static pod itself.
apt-get install -y etcd-client

# The cert paths come from the etcd static pod manifest. If you forget them,
# read them off: grep -E "cert-file|key-file|trusted-ca" /etc/kubernetes/manifests/etcd.yaml
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/etcd-backup.db

# Always verify the snapshot.
ETCDCTL_API=3 etcdctl --write-out=table snapshot status /opt/etcd-backup.db
```

Now destroy something and restore:

```bash
# Back on WSL — create marker resources, then delete them.
kubectl create ns restoretest
kubectl -n restoretest create deploy marker --image=nginx
kubectl -n restoretest get all

kubectl delete ns restoretest
```

```bash
# On cka-cp1 as root. Restore to a NEW directory — never over the live one.
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# Point the etcd static pod at the restored data. Edit the hostPath volume
# whose name is etcd-data: change /var/lib/etcd to /var/lib/etcd-restored.
vi /etc/kubernetes/manifests/etcd.yaml

# The kubelet notices the manifest change and restarts etcd automatically.
# The API server will be briefly unavailable — expected.
crictl ps | grep etcd
```

```bash
# Back on WSL, after a minute
kubectl -n restoretest get all      # the deleted namespace is back
```

**The step people miss is editing the manifest.** Restoring the data directory alone does nothing — etcd is still reading the old path.

### Exercise 6.2 — Cluster upgrade, v1.34 → v1.35 🔴 on-node

Snapshot all three VMs first. Do this twice: once following along, once from memory.

**Control plane:**

```bash
ssh cka-cp1 && sudo -i
```

```bash
# STEP THAT EVERYONE MISSES: pkgs.k8s.io repos are per-minor-version.
# You must change the repo URL before apt can even see 1.35 packages.
sed -i 's|v1.34|v1.35|' /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg --yes
apt-get update

# Find the exact available patch version.
apt-cache madison kubeadm | head -5

# Upgrade kubeadm itself first — it drives everything else.
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.35.0-1.1      # substitute the real version
apt-mark hold kubeadm

# Shows what will change and verifies the upgrade path is valid.
kubeadm upgrade plan

# Upgrades the control plane static pods (apiserver, scheduler,
# controller-manager, etcd) and renews certs.
kubeadm upgrade apply v1.35.0
```

```bash
# Drain BEFORE upgrading the kubelet. --ignore-daemonsets is always needed
# because DaemonSet pods can't be rescheduled elsewhere.
kubectl drain cka-cp1 --ignore-daemonsets

apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.35.0-1.1 kubectl=1.35.0-1.1
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

kubectl uncordon cka-cp1
kubectl get nodes        # cka-cp1 now v1.35.0
```

**Workers** — note `kubeadm upgrade node`, not `apply`:

```bash
ssh cka-w1 && sudo -i
sed -i 's|v1.34|v1.35|' /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg --yes
apt-get update
apt-mark unhold kubeadm && apt-get install -y kubeadm=1.35.0-1.1 && apt-mark hold kubeadm

kubeadm upgrade node        # updates the local kubelet config only
```

```bash
# Drain from WSL, then upgrade the kubelet on the node, then uncordon.
kubectl drain cka-w1 --ignore-daemonsets --delete-emptydir-data
```

```bash
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.35.0-1.1 kubectl=1.35.0-1.1
apt-mark hold kubelet kubectl
systemctl daemon-reload && systemctl restart kubelet
```

```bash
kubectl uncordon cka-w1
```

Repeat for `cka-w2`. Then **snapshot as `clean-v135`** and update your WSL kubectl to 1.35 to stay in skew.

### Exercise 6.3 — Node lifecycle

```bash
kubectl cordon cka-w1                  # no new pods; existing ones stay
kubectl run test --image=nginx         # lands elsewhere
kubectl uncordon cka-w1

# Drain evicts as well as cordons. Try it without the flag first to see
# the DaemonSet error, so you recognize it later.
kubectl drain cka-w1
kubectl drain cka-w1 --ignore-daemonsets --delete-emptydir-data --force
kubectl uncordon cka-w1
```

### Exercise 6.4 — Join a node from scratch

```bash
# On cka-w2: tear it out of the cluster entirely.
ssh cka-w2 && sudo kubeadm reset -f && sudo rm -rf /etc/cni/net.d
exit
```

```bash
kubectl delete node cka-w2

# Tokens expire after 24h. This regenerates one — memorize this command.
ssh cka-cp1 'sudo kubeadm token create --print-join-command'
```

Run the printed command on `cka-w2` with `sudo`, then confirm it returns `Ready`.

### Exercise 6.5 — Certificates

```bash
ssh cka-cp1
sudo kubeadm certs check-expiration    # which certs expire when
sudo kubeadm certs renew all
```

Expired certs present as `Unable to connect to the server: x509`. Worth seeing the healthy output so you recognize the unhealthy one.

### Drills

Library items **28, 29, 30**, plus **7, 8, 9, 11** (control plane breakage).

### Done when

- [ ] I can back up and restore etcd, including the manifest edit
- [ ] I've upgraded the full cluster twice, the second time unaided
- [ ] I remember that the apt repo URL must change per minor version
- [ ] I know `upgrade apply` (control plane) vs `upgrade node` (workers)
- [ ] I can regenerate a join token and re-add a node
- [ ] I can check cert expiry and renew

---

## Week 7 — RBAC, Helm, Kustomize, CRDs

### Exercise 7.1 — RBAC

```bash
kubectl create ns dev
kubectl -n dev create sa deployer

# Role = namespaced. ClusterRole = cluster-wide.
kubectl -n dev create role pod-reader --verb=get,list,watch --resource=pods
kubectl -n dev create rolebinding deployer-reads \
  --role=pod-reader --serviceaccount=dev:deployer

# auth can-i --as is THE verification tool. Use it constantly.
kubectl auth can-i list pods --as=system:serviceaccount:dev:deployer -n dev     # yes
kubectl auth can-i list pods --as=system:serviceaccount:dev:deployer -n default # no
kubectl auth can-i delete pods --as=system:serviceaccount:dev:deployer -n dev   # no
```

Work through all four combinations, since knowing which to use is the actual exam skill:

```bash
# ClusterRole + RoleBinding — cluster-scoped permissions, granted in ONE namespace.
# This is the combination people forget exists, and it's frequently the answer.
kubectl create clusterrole secret-reader --verb=get,list --resource=secrets
kubectl -n dev create rolebinding deployer-secrets \
  --clusterrole=secret-reader --serviceaccount=dev:deployer

kubectl auth can-i list secrets --as=system:serviceaccount:dev:deployer -n dev      # yes
kubectl auth can-i list secrets --as=system:serviceaccount:dev:deployer -n default  # no

# ClusterRole + ClusterRoleBinding — everywhere
kubectl create clusterrolebinding deployer-all \
  --clusterrole=secret-reader --serviceaccount=dev:deployer
kubectl auth can-i list secrets --as=system:serviceaccount:dev:deployer -n default  # now yes
```

Then verify from inside a pod using the mounted token:

```bash
kubectl -n dev run probe --image=nginx --overrides='{"spec":{"serviceAccountName":"deployer"}}'
kubectl -n dev exec probe -- ls /var/run/secrets/kubernetes.io/serviceaccount/
```

### Exercise 7.2 — Helm

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/nginx

# ALWAYS template before installing — it renders locally with no cluster changes.
helm template myweb bitnami/nginx | head -40

helm install myweb bitnami/nginx --set replicaCount=2
helm list
helm status myweb

# Upgrade, then roll back — both are exam verbs.
helm upgrade myweb bitnami/nginx --set replicaCount=4
helm history myweb
helm rollback myweb 1
helm history myweb

helm get values myweb
helm uninstall myweb
```

```bash
# Values files, since --set doesn't scale
cat <<'YAMLEOF' > values.yaml
replicaCount: 3
service:
  type: ClusterIP
YAMLEOF
helm install myweb bitnami/nginx -f values.yaml --dry-run --debug
```

### Exercise 7.3 — Kustomize

```bash
mkdir -p ~/kustomize-lab/{base,overlays/prod}
cd ~/kustomize-lab

kubectl create deploy web --image=nginx --replicas=1 $do > base/deployment.yaml
kubectl create svc clusterip web --tcp=80:80 $do > base/service.yaml

cat <<'YAMLEOF' > base/kustomization.yaml
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  app: web
YAMLEOF

cat <<'YAMLEOF' > overlays/prod/kustomization.yaml
resources:
  - ../../base
namePrefix: prod-
replicas:
  - name: web
    count: 5
configMapGenerator:
  - name: app-config
    literals:
      - ENV=production
YAMLEOF

# Render without applying — always check this first.
kubectl kustomize overlays/prod

# Apply with -k
kubectl apply -k overlays/prod
kubectl get deploy,svc,cm
```

Note the hash suffix Kustomize appends to generated ConfigMaps — that's deliberate, so a config change forces a rollout.

### Exercise 7.4 — CRDs and operators

```bash
# Install an operator. cert-manager brings CRDs plus a controller.
helm repo add jetstack https://charts.jetstack.io && helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true

kubectl get crd | grep cert-manager
kubectl api-resources | grep cert-manager

# A CRD extends the API — kubectl explain works on custom types too.
kubectl explain certificate.spec
```

```bash
# Create a custom resource and watch the controller act on it.
cat <<'YAMLEOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: {name: selfsigned}
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: {name: test-cert, namespace: default}
spec:
  secretName: test-cert-tls
  issuerRef: {name: selfsigned, kind: ClusterIssuer}
  commonName: test.lab.local
  dnsNames: [test.lab.local]
YAMLEOF

# The controller reconciles your CR into an actual Secret. That reconcile
# loop IS the operator pattern.
kubectl get certificate
kubectl get secret test-cert-tls
```

Also write a bare CRD yourself so you've seen the schema:

```bash
cat <<'YAMLEOF' | kubectl apply -f -
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: {name: widgets.lab.example.com}
spec:
  group: lab.example.com
  scope: Namespaced
  names: {plural: widgets, singular: widget, kind: Widget}
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size: {type: string}
YAMLEOF

kubectl get crd widgets.lab.example.com
```

### Exercise 7.5 — Extension interfaces 🔴 on-node

```bash
ssh cka-cp1
sudo ls /etc/cni/net.d/              # CNI config — deleting this is drill 12
sudo cat /etc/cni/net.d/*.conflist
sudo crictl info | head -30          # CRI — the runtime kubelet talks to
kubectl get csidrivers               # CSI — storage drivers
```

### Drills

Library items **26, 27**, plus **12**.

### Done when

- [ ] I can pick the right Role/ClusterRole + binding combination for a scenario
- [ ] I use `auth can-i --as` to verify rather than guessing
- [ ] I can install, upgrade, roll back and uninstall a Helm release
- [ ] I can build a base + overlay and render it with `kubectl kustomize`
- [ ] I can install a CRD, create a CR, and explain the reconcile loop

---

## Week 8 — Troubleshooting Boot Camp (30%)

**The week that decides your score.** No new material — pure drill repetition. Aim for 6–8 drills per session, blind, timed.

### Your triage checklist

Write this out and refine it all week. A reasonable starting version:

```
1. kubectl get nodes              — all Ready?
2. kubectl get pods -A            — anything not Running?
3. kubectl describe pod <x>       — read the Events at the bottom FIRST
4. kubectl logs <x> [--previous]  — app-level errors
5. Service issue? -> endpoints, then targetPort, then DNS
6. Node issue?    -> ssh, systemctl status kubelet, journalctl -u kubelet
7. Control plane? -> ssh cp, crictl ps -a, read the static pod manifest
```

### Exercise 8.1 — Node-level tooling 🔴 on-node

```bash
ssh cka-w1
```

```bash
# The kubelet is a systemd unit, not a pod. When the cluster can't see a node,
# this is where you go — kubectl will tell you nothing useful.
systemctl status kubelet
journalctl -u kubelet -f
journalctl -u kubelet --since "10 minutes ago" --no-pager | tail -50

# crictl talks to containerd directly. Essential when the API server is down,
# because kubectl is useless then.
sudo crictl ps -a
sudo crictl logs <container-id>
sudo crictl images

# Kubelet config and cert locations — know these paths cold.
sudo cat /var/lib/kubelet/config.yaml
ls /etc/kubernetes/            # kubelet.conf, bootstrap-kubelet.conf
ls /var/lib/kubelet/pki/
```

### Exercise 8.2 — Control plane down 🔴 on-node

The scenario worth rehearsing most, because your usual tools stop working.

```bash
ssh cka-cp1 && sudo -i

# Break the API server deliberately.
cp /etc/kubernetes/manifests/kube-apiserver.yaml /root/apiserver.bak
sed -i 's|--etcd-servers=.*|--etcd-servers=https://127.0.0.1:9999|' \
  /etc/kubernetes/manifests/kube-apiserver.yaml
```

Now `kubectl` fails entirely. Work the problem with what's left:

```bash
# crictl still works — containerd doesn't need the API server.
crictl ps -a | grep apiserver
crictl logs $(crictl ps -a --name kube-apiserver -q | head -1)

# The kubelet logs its attempts to start the static pod.
journalctl -u kubelet --no-pager | tail -30

# Read the manifest and find the bad flag.
cat /etc/kubernetes/manifests/kube-iserver.yaml | grep etcd

cp /root/apiserver.bak /etc/kubernetes/manifests/kube-apiserver.yaml
```

Recovery takes 1–2 minutes. **Don't panic and start changing other things** — that's how a one-flag problem becomes a rebuild.

### Exercise 8.3 — Resource monitoring

```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl top pods -A --sort-by=cpu

# Container output streams — note the flags.
kubectl logs <pod> -c <container>       # multi-container pod
kubectl logs <pod> --previous           # the CRASHED instance, not the current one
kubectl logs -l app=web --tail=50       # all pods matching a label
kubectl logs <pod> --since=10m
```

`--previous` is the one people forget. On a CrashLoopBackOff the current container has barely started; the error is in the previous one.

### Exercise 8.4 — Full drill rotation

Work the library in §4 of the guide. Suggested schedule:

| Session | Drills | Focus |
|---|---|---|
| 1 | 1–6 | Node and kubelet |
| 2 | 7–11 | Control plane |
|–17 | Networking |
| 4 | 18–23 | Workloads |
| 5 | 24–27 | Storage and RBAC |
| 6 | 28–30 | Lifecycle |
| 7 | Random 10, blind | Mixed, timed |
| 8 | Random 10, blind | Mixed, timed |

**Target by session 8: diagnose in 3 minutes, fix in 7.**

### Done when

- [ ] My triage checklist is written and I follow it automatically
- [ ] I can work a dead control plane using only `crictl` and `journalctl`
- [ ] I reach for `--previous` on any crashlooping pod
- [ ] I've done all 30 drills, and the last 20 b ] My average is under 10 minutes per drill

---

## Week 9 — Killer.sh and Timed Practice

### Exercise 9.1 — Killer.sh session 1

Activate from your Linux Foundation portal. **Take it cold, under real conditions** — 2 hours, one monitor, only the allowed docs.

Expect roughly 40–60%. The simulator is deliberately harder than the real exam; this is normal and widely reported. Don't read it as a verdict.

You have **36 hours of access** after activation, and the environment resets on demand. So:

1. cold, timed. Record your score.
2. Work through every solution, including tasks you got right.
3. Re-run the same session and aim to complete it in half the time.
4. Re-run once more, targeting 100%.

### Exercise 9.2 — Weak-domain repair

Sort your missed tasks by domain and attack the worst one. Re-run the relevant drills from Week 8.

### Exercise 9.3 — Killercoda scenarios

Free CKA scenarios at `killercoda.com/killer-shell-cka`. Do them timed, without notes.

### Exercise 9.4 — Environment rehea```bash
# Rehearse the exam's SSH pattern explicitly. On exam day you start on a
# base host with NO tools and hop per-task.
ssh cka-cp1
# ...do the task...
exit
ssh cka-w1
# ...do the task...
exit
```

Practice on one monitor, at the resolution you'll use, with only the four allowed doc sites open.

### Done when

- [ ] Killer.sh session 1 taken cold and fully reviewed
- [ ] Every missed task re-done from scratch
- [ ] Weakest domain re-drilled
- [ ] Two sessions completed on a single monitor

---

## Week 10 — Final Preparation

### Exercise 10.1 — Killer.sh session 2

**Take session B 2–3 days before your exam.** Different questions. Target 70%+ within the time limit — on a test harder than the real thing, that's a comfortable margin.

### Exercise 10.2 — Speed audit

Time yourself on the fundamentals. If any of these exceed the target, drill it:

| Task | Target |
|---|---|
| Deployment + service + scale | 90s |
| ConfigMap and Secret into a pod, 3 ways | 3 min |
| RBAC: SA + Role + binding, vermin |
| PV + PVC + pod mount | 3 min |
| NetworkPolicy default-deny + one allow | 4 min |
| etcd backup and verify | 4 min |
| Diagnose a NotReady node | 3 min |
| Diagnose a service with no endpoints | 3 min |
| Ingress with host rules | 3 min |
| Gateway + HTTPRoute | 5 min |

### Exercise 10.3 — Documentation navigation

Find each of these in under 30 seconds, using only the allowed sites:

- etcd backup and restore procedure
- kubeadm upgrade steps
- NetworkPolicy example manifests
- PersistentVolume nfiguration reference
- HTTPRoute specification (`gateway-api.sigs.k8s.io`)
- Helm chart install and rollback (`helm.sh/docs`)
- RBAC role examples
- Pod security and scheduling reference

Bookmark all eight. You can use bookmarks in the exam.

### Exercise 10.4 — Two full mock exams

Build your own: pick 17 tasks across the domains at the official weights, set a 2-hour timer, and work them with only the allowed docs on one screen. Mark honestly. Partial credit counts.

### Exercise 10.5 — Logistics

- PSI system check run and passed
- [ ] Photo ID name matches your LF profile exactly
- [ ] Workspace cleared, walls bare, room private
- [ ] Wired ethernet if available
- [ ] Antivirus/firewall exceptions for the secure browser
- [ ] Exam scheduled; plan to check in 15 minutes early

### Exam-day reminders

1. Read all tasks first; triage by weight and confidence
2. SSH to the correct host **every time**
3. Hard-stop any task at 7 minutes; flag and return
4. Generate YAML with `$do`; don't hand-write it
5. Verify completed tasks if time allows
6. **66% passes.** Skipping one impossible task is a valid strategy

### Done when

- [ ] Killer.sh session 2 at 70%+
- [ ] All speed-audit targets met
- [ ] Two self-built mocks completed and marked
- [ ] Logistics checklist complete

---

## Appendix — Daily Warm-Up (10 minutes)

Run this before every session from Week 2 onward. It keeps the fundamentals fast while you work on harder material.

```bash
# 1. Generate five resources imperatively, from memory
k create dloy warmup --image=nginx --replicas=3 $do > /dev/null
k expose deploy warmup --port=80 $do > /dev/null
k create cm warmup --from-literal=k=v $do > /dev/null
k create role warmup --verb=get --resource=pods $do > /dev/null
k create job warmup --image=busybox $do -- echo hi > /dev/null

# 2. Cluster health sweep
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed
kubectl top nodes

# 3. One random drill from the library, timed
```

