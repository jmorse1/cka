# Homelab Observability on Proxmox — Setup Guide

Oct 27, 2026 · @Jason Morse

## Overview

This is the Azure design shrunk to one Proxmox host: Garage replaces Blob Storage, Mimir, Loki, and Tempo run in single-binary (monolithic) mode so no Kafka is needed, and self-hosted Grafana OSS replaces Azure Managed Grafana.

The architecture and the habits carry over: collectors enrich and forward, specialized engines store, object storage holds everything durable, Grafana only visualizes, and everything is declared in Git. What doesn't carry over is fault tolerance, because every VM shares one physical machine and one disk.

| Azure design | Proxmox homelab | Why |
| --- | --- | --- |
| Observability AKS cluster (3 zones) | 2 new worker VMs from your golden image, labeled for observability | Your current 2 vCPU / 4 GB nodes can't hold the stack |
| Azure Blob Storage | Garage (S3-compatible) in its own VM | MinIO's community edition was archived in April 2026 |
| Mimir / Loki / Tempo distributed + Kafka | Monolithic mode for each, no Kafka | Monolithic modes don't use Kafka; distributed 3.x modes do |
| Azure Managed Grafana | Grafana OSS (Helm), SQLite or Postgres | No managed option on-prem |
| Private Link Services | ClusterIP services + one OTel gateway + NetworkPolicies | One cluster, one network; isolation by namespace |
| Key Vault + workload identity | Kubernetes Secrets (sealed or SOPS-encrypted in Git) | Garage uses access keys, not identities |
| Azure Monitor for platform health | Proxmox metric export + a few exporters into the same stack | Plus an outside "is it up" check |
| GZRS / region recovery | Nightly copy of buckets to the NAS or an external disk/cloud | One host means one failure domain |

&#91;embedded content: Homelab architecture · one host, one cluster, Garage for durable data\]

Agents on every node send to one gateway, which fans out to Mimir, Loki, and Tempo on the new obs workers; they persist to Garage, and only the nightly off-host copy survives a drive failure.

## Sizing

Add two 4 vCPU / 12 GB worker VMs and one small Garage VM; the existing 2 vCPU / 4 GB nodes keep running your apps plus a collector each.

A 4 GB node loses roughly 1 GB to the OS, kubelet, and system pods, which leaves too little for an ingester holding recent data in memory. Putting the stack on dedicated nodes also stops a memory spike in Mimir from evicting your apps.

**Starting budget (approximate, homelab scale: a few nodes, under \~200k active series, a few GB of logs a day):**

| Workload | Where | Request | Limit |
| --- | --- | --- | --- |
| Mimir (monolithic) | obs workers | 0.5 vCPU / 1.5 GB | 2 GB |
| Loki (monolithic) | obs workers | 0.5 vCPU / 1 GB | 2 GB |
| Tempo (monolithic) | obs workers | 0.5 vCPU / 1 GB | 2 GB |
| Grafana | obs workers | 0.2 vCPU / 256 MB | 512 MB |
| OTel gateway | obs workers | 0.2 vCPU / 256 MB | 512 MB |
| OTel agent (DaemonSet) | every node | 0.1 vCPU / 128 MB | 256 MB |
| Loki chunk + results caches | obs workers | turn off, or 256 MB each | 512 MB |
| Garage | its own VM | 1 vCPU / 1 GB | 2 GB VM |

**VMs to add:**

| VM | vCPU | RAM | Disk |
| --- | --- | --- | --- |
| k8s-obs-1 | 4 | 12 GB | 60 GB (OS + PVCs) |
| k8s-obs-2 | 4 | 12 GB | 60 GB |
| garage-1 | 2 | 2 GB | 32 GB OS + separate data disk (200 GB+) |

That's 10 vCPU and 26 GB of new allocation. Check the Proxmox host's free RAM first (**Datacenter > host > Summary**); if it's short, start with one obs worker at 16 GB and accept that the stack restarts when that VM reboots. vCPU overcommit is fine on Proxmox; RAM overcommit is not, so don't rely on ballooning for these VMs.

## Step 1: Add observability worker nodes

Clone two VMs from the golden image, resize them, join them to the cluster, then label and taint them so only the observability stack lands there.

**1a. Clone and resize on the Proxmox host** (replace `9000` with your template's VM ID):

```bash
for i in 1 2; do
  ID=$((210 + i))
  qm clone 9000 $ID --name k8s-obs-$i --full true
  qm set $ID --cores 4 --memory 12288 --balloon 0
  qm set $ID --ipconfig0 ip=192.168.1.$ID/24,gw=192.168.1.1   # if the template uses cloud-init
  qm resize $ID scsi0 60G
  qm set $ID --onboot 1
  qm start $ID
done
```

If the golden image isn't a Proxmox template with cloud-init, clone it from the GUI (**Right-click > Clone > Full clone**) and set hostname and IP inside the VM. Make sure each clone gets a fresh machine-id (`rm /etc/machine-id && systemd-machine-id-setup`) or the nodes can collide in Kubernetes and DHCP.

**1b. Join the cluster.** Use whichever matches how the cluster was built:

```bash
# kubeadm: on a control-plane node, print a join command, then run it on each new VM
kubeadm token create --print-join-command

# k3s: on each new VM
curl -sfL https://get.k3s.io | K3S_URL=https://<server-ip>:6443 K3S_TOKEN=<node-token> sh -s - agent
```

**1c. Label and taint**

```bash
for n in k8s-obs-1 k8s-obs-2; do
  kubectl label node $n node-role.kubernetes.io/observability="" workload=observability
  kubectl taint node $n workload=observability:NoSchedule
done
```

Every observability chart below gets this pair of settings:

```yaml
nodeSelector:
  workload: observability
tolerations:
  - key: workload
    value: observability
    effect: NoSchedule
```

The OTel agent DaemonSet gets the toleration but no nodeSelector, so it runs on every node.

Take a Proxmox snapshot of both new VMs once they're joined and healthy; it's your fast rollback while experimenting.

## Step 2: Persistent volumes

Use node-local disks (local-path provisioner) for the stack's working data, and keep NFS from the NAS VM out of the write path.

Mimir, Loki, and Tempo only keep a write-ahead log and a few hours of recent data on local volumes; everything older lives in Garage. That working data is fsync-heavy, and TSDB and WAL files on NFS are a well-known source of corruption and slow flushes. Local disks are faster and simpler.

```bash
# k3s ships local-path already (StorageClass "local-path"). For kubeadm:
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl get storageclass
```

| Volume | StorageClass | Size |
| --- | --- | --- |
| Mimir WAL + TSDB head | local-path | 10 GB |
| Loki WAL + index cache | local-path | 10 GB |
| Tempo WAL | local-path | 10 GB |
| Grafana (SQLite DB, plugins) | local-path | 2 GB |

A local-path volume pins its pod to that node. That's acceptable here: if `k8s-obs-1` is down, its pods wait for it, and no durable data is lost because blocks, chunks, and traces are already in Garage.

Where the NAS VM does help: as the target for backups in Step 10, and optionally as Garage's data disk (Step 3, option B).

## Step 3: Object storage with Garage (the blob equivalent)

Run Garage, a single static Rust binary that speaks S3, in its own small VM with replication factor 1, and give each of Mimir, Loki, and Tempo its own bucket and access key.

**Why Garage and not MinIO.** MinIO's community edition stripped its admin console in May 2025, stopped publishing binaries and images in October 2025, and was archived as read-only on April 25, 2026. Garage, SeaweedFS, and RustFS are the common replacements; Garage is the leanest fit for one node, and people already run it under Loki and Tempo.

| Option | Fits a homelab because | Trade-off |
| --- | --- | --- |
| **Garage** (recommended) | One binary, low RAM, simple CLI | AGPL; fewer S3 extras (no versioning UI) |
| SeaweedFS | Has a web admin UI, very mature | More moving parts (master, volume, filer) |
| RustFS | MinIO-like UX | Younger project; treat as a pilot |

**Placement.** Option A (recommended): a dedicated `garage-1` VM with a separate virtual data disk. Option B: run Garage as a container on the NAS VM. A gives you a clean snapshot and resource boundary; B saves a VM. Either way the data lands on the Proxmox host's internal drive, so Step 10's off-box backup matters.

**3a. Install** (on `garage-1`, Debian or Ubuntu; check the Garage site for the current release, v2.3.0 at time of writing):

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin garage
sudo mkdir -p /var/lib/garage/meta /srv/garage/data   # /srv on the data disk
sudo chown -R garage:garage /var/lib/garage /srv/garage
wget https://garagehq.deuxfleurs.fr/_releases/v2.3.0/x86_64-unknown-linux-musl/garage -O /tmp/garage
sudo install -m 755 /tmp/garage /usr/local/bin/garage
```

**3b. `/etc/garage.toml`**

```toml
metadata_dir = "/var/lib/garage/meta"
data_dir     = "/srv/garage/data"
db_engine    = "sqlite"
replication_factor = 1          # single node
compression_level  = 1

rpc_bind_addr   = "[::]:3901"
rpc_public_addr = "192.168.1.220:3901"
rpc_secret      = "<openssl rand -hex 32>"

[s3_api]
s3_region     = "garage"
api_bind_addr = "[::]:3900"

[admin]
api_bind_addr = "[::]:3903"
admin_token   = "<openssl rand -base64 32>"
metrics_token = "<openssl rand -base64 32>"
```

Run it under a systemd unit as the `garage` user (`ExecStart=/usr/local/bin/garage server`), then start it.

**3c. Layout, buckets, keys**

```bash
garage status                                   # copy the node ID
garage layout assign -z homelab -c 200G <node-id>
garage layout apply --version 1                 # Garage refuses writes until a layout is applied

for b in mimir-blocks mimir-ruler mimir-alertmanager loki-chunks loki-ruler tempo-traces; do
  garage bucket create $b
done

garage key create mimir-key
garage key create loki-key
garage key create tempo-key

for b in mimir-blocks mimir-ruler mimir-alertmanager; do garage bucket allow --read --write $b --key mimir-key; done
for b in loki-chunks loki-ruler;                     do garage bucket allow --read --write $b --key loki-key; done
garage bucket allow --read --write tempo-traces --key tempo-key

garage key info mimir-key --show-secret          # repeat per key; store in Kubernetes Secrets
```

**3d. Kubernetes Secrets** (one per namespace, created before the Helm installs):

```bash
kubectl create ns mimir; kubectl create ns loki; kubectl create ns tempo
kubectl -n mimir create secret generic s3-creds --from-literal=AWS_ACCESS_KEY_ID=<id> --from-literal=AWS_SECRET_ACCESS_KEY=<secret>
# same for loki and tempo with their own keys
```

Give `garage-1` a DNS name (for example `garage.lab`) in your router or Pi-hole. All three clients must use **path-style** addressing, because Garage won't resolve `bucket.garage.lab` virtual hosts without extra DNS setup.

## Step 4: Mimir (metrics), monolithic

Run Mimir as one StatefulSet with `-target=all`, ingest storage off, and Garage as the S3 backend; the `mimir-distributed` chart is overkill at this size and defaults to Kafka.

In Mimir 3.x the binary only uses Kafka when ingest storage is explicitly enabled, and the classic architecture is still supported. That's what makes Kafka unnecessary here.

**4a. Config** (`mimir-config` ConfigMap):

```yaml
target: all
multitenancy_enabled: true
server:
  http_listen_port: 8080
  grpc_listen_port: 9095

common:
  storage:
    backend: s3
    s3:
      endpoint: garage.lab:3900
      region: garage
      insecure: true
      bucket_lookup_type: path
      access_key_id: ${AWS_ACCESS_KEY_ID}
      secret_access_key: ${AWS_SECRET_ACCESS_KEY}

blocks_storage:
  s3: { bucket_name: mimir-blocks }
  tsdb: { dir: /data/tsdb }
  bucket_store: { sync_dir: /data/tsdb-sync }
ruler_storage:
  s3: { bucket_name: mimir-ruler }
alertmanager_storage:
  s3: { bucket_name: mimir-alertmanager }

ingest_storage:
  enabled: false
ingester:
  push_grpc_method_enabled: true
  ring:
    replication_factor: 1
store_gateway:
  sharding_ring: { replication_factor: 1 }
compactor:
  data_dir: /data/compactor

memberlist:
  join_members: [mimir-headless.mimir.svc:7946]

limits:
  compactor_blocks_retention_period: 90d
  max_global_series_per_user: 300000
  otel_metric_suffixes_enabled: true
```

**4b. Workload** (the essentials of the StatefulSet; one replica):

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: mimir, namespace: mimir }
spec:
  serviceName: mimir-headless
  replicas: 1
  selector: { matchLabels: { app: mimir } }
  template:
    metadata: { labels: { app: mimir } }
    spec:
      nodeSelector: { workload: observability }
      tolerations: [{ key: workload, value: observability, effect: NoSchedule }]
      containers:
        - name: mimir
          image: grafana/mimir:<pin a 3.x tag>
          args: ["-config.file=/etc/mimir/mimir.yaml", "-config.expand-env=true"]
          envFrom: [{ secretRef: { name: s3-creds } }]
          ports:
            - { name: http, containerPort: 8080 }
            - { name: grpc, containerPort: 9095 }
            - { name: memberlist, containerPort: 7946 }
          readinessProbe: { httpGet: { path: /ready, port: http } }
          resources:
            requests: { cpu: 500m, memory: 1536Mi }
            limits: { memory: 2Gi }
          volumeMounts:
            - { name: config, mountPath: /etc/mimir }
            - { name: data, mountPath: /data }
      volumes: [{ name: config, configMap: { name: mimir-config } }]
  volumeClaimTemplates:
    - metadata: { name: data }
      spec: { storageClassName: local-path, accessModes: [ReadWriteOnce], resources: { requests: { storage: 10Gi } } }
```

Add two Services: `mimir-headless` (clusterIP: None, port 7946) and `mimir` (ClusterIP, port 8080).

**Endpoints you'll use:** OTLP write `http://mimir.mimir.svc:8080/otlp/v1/metrics`, PromQL read `http://mimir.mimir.svc:8080/prometheus`. Both need an `X-Scope-OrgID` header because multitenancy is on; use one tenant, `homelab`.

**Check it:** `kubectl -n mimir logs sts/mimir | grep -i error` is quiet, and within about 2 hours the first block shows up in `garage bucket info mimir-blocks`.

## Step 5: Loki (logs), monolithic

Install the `loki` chart from `grafana-community` with `deploymentMode: Monolithic`, one replica, S3 pointed at Garage with path-style addressing.

Two chart changes to know: the OSS chart moved to the `grafana-community` repo on March 16, 2026, and from community chart 12.0.0 the old `SingleBinary` mode is named `Monolithic`. Simple Scalable mode is being deprecated before Loki 4.0, so skip it.

```yaml
# loki-values.yaml
deploymentMode: Monolithic

loki:
  auth_enabled: true
  commonConfig:
    replication_factor: 1
  schemaConfig:
    configs:
      - from: "2026-11-01"
        store: tsdb
        object_store: s3
        schema: v13
        index: { prefix: loki_index_, period: 24h }
  storage:
    type: s3
    bucketNames:
      chunks: loki-chunks
      ruler: loki-ruler
    s3:
      endpoint: http://garage.lab:3900
      region: garage
      s3ForcePathStyle: true
      insecure: true
      accessKeyId: ${AWS_ACCESS_KEY_ID}
      secretAccessKey: ${AWS_SECRET_ACCESS_KEY}
  limits_config:
    allow_structured_metadata: true
    retention_period: 336h          # 14 days
  compactor:
    retention_enabled: true
    delete_request_store: s3

monolithic:
  replicas: 1
  extraArgs: ["-config.expand-env=true"]
  extraEnvFrom: [{ secretRef: { name: s3-creds } }]
  persistence: { storageClass: local-path, size: 10Gi }
  nodeSelector: { workload: observability }
  tolerations: [{ key: workload, value: observability, effect: NoSchedule }]
  resources: { requests: { cpu: 500m, memory: 1Gi }, limits: { memory: 2Gi } }

chunksCache:  { enabled: false }    # saves ~8 GB of default memcached requests
resultsCache: { enabled: false }
lokiCanary:   { enabled: false }
gateway:      { enabled: false }    # the OTel gateway and Grafana talk to Loki directly
minio:        { enabled: false }
```

```bash
helm repo add grafana-community https://grafana-community.github.io/helm-charts && helm repo update
helm install loki grafana-community/loki -n loki -f loki-values.yaml
```

The monolithic values block name has shifted between chart versions (`singleBinary` before 12.0.0). Run `helm show values grafana-community/loki | less` for the version you pin and adjust the key if needed. The default memcached caches request several GB, which is why they're off here.

**Endpoints:** OTLP write `http://loki.loki.svc:3100/otlp/v1/logs`, query `http://loki.loki.svc:3100`, header `X-Scope-OrgID: homelab`.

## Step 6: Tempo (traces), monolithic

Install the single-binary `tempo` chart from `grafana-community` with S3 on Garage; monolithic Tempo needs no Kafka in either 2.x or 3.x.

In Tempo 3.x, Kafka sits only in the microservices write path; in monolithic mode the distributor hands spans to the live-store in-process. The community `tempo` chart is the single-binary one (`tempo-distributed` is the microservices chart).

```yaml
# tempo-values.yaml
tempo:
  multitenancyEnabled: true
  retention: 168h                    # 7 days
  extraArgs:
    config.expand-env: true
  extraEnvFrom:
    - secretRef: { name: s3-creds }
  storage:
    trace:
      backend: s3
      s3:
        bucket: tempo-traces
        endpoint: garage.lab:3900
        region: garage
        insecure: true
        forcepathstyle: true
        access_key: ${AWS_ACCESS_KEY_ID}
        secret_key: ${AWS_SECRET_ACCESS_KEY}
  receivers:
    otlp:
      protocols:
        grpc: { endpoint: 0.0.0.0:4317 }
        http: { endpoint: 0.0.0.0:4318 }
  resources: { requests: { cpu: 500m, memory: 1Gi }, limits: { memory: 2Gi } }

persistence:
  enabled: true
  storageClassName: local-path
  size: 10Gi

nodeSelector: { workload: observability }
tolerations: [{ key: workload, value: observability, effect: NoSchedule }]
```

```bash
helm install tempo grafana-community/tempo -n tempo -f tempo-values.yaml
helm show values grafana-community/tempo | less     # confirm keys for the version you pin
```

If you pin a Tempo 3.x image, move any retention setting to wherever your version's config reference puts it (3.x replaced the compactor with a backend scheduler and worker).

**Endpoints:** OTLP write `tempo.tempo.svc:4317` (gRPC), query `http://tempo.tempo.svc:3200`, header `X-Scope-OrgID: homelab`.

## Step 7: OpenTelemetry collectors and the gateway

Install the `opentelemetry-collector` chart three times: an agent DaemonSet on every node, one cluster-level collector, and one gateway that is the only thing allowed to write to Mimir, Loki, and Tempo.

Keeping the gateway even in one cluster preserves the Azure design's shape: agents never know backend addresses, tenancy is set in one place, and you can later point a second cluster (or that AWS environment) at the same gateway.

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts && helm repo update
kubectl create ns otel
```

**7a. Agent (DaemonSet)** — chart presets do most of the work:

```yaml
# otel-agent.yaml
mode: daemonset
image: { repository: otel/opentelemetry-collector-contrib }
tolerations: [{ key: workload, value: observability, effect: NoSchedule }]
presets:
  logsCollection:       { enabled: true }    # filelog on /var/log/pods
  kubeletMetrics:       { enabled: true }
  kubernetesAttributes: { enabled: true }    # k8sattributes + RBAC
  hostMetrics:          { enabled: true }
config:
  processors:
    resource:
      attributes:
        - { key: k8s.cluster.name, value: homelab, action: upsert }
        - { key: deployment.environment.name, value: homelab, action: upsert }
  exporters:
    otlp/gateway:
      endpoint: otel-gateway-opentelemetry-collector.otel.svc:4317
      tls: { insecure: true }
  service:
    pipelines:
      logs:    { exporters: [otlp/gateway] }
      metrics: { exporters: [otlp/gateway] }
      traces:  { exporters: [otlp/gateway] }
resources: { requests: { cpu: 100m, memory: 128Mi }, limits: { memory: 256Mi } }
```

Add a `prometheus` receiver with Kubernetes pod discovery to scrape apps that expose `/metrics`, and add `resource` to each pipeline's processors list.

**7b. Cluster collector (1 replica)** — `mode: deployment`, `replicaCount: 1`, presets `clusterMetrics` and `kubernetesEvents`, same exporter. One replica avoids duplicate cluster series.

**7c. Gateway (1 replica)** — `mode: deployment`, on the obs nodes, with the tenant set once:

```yaml
# otel-gateway.yaml (config section)
config:
  receivers:
    otlp:
      protocols: { grpc: { endpoint: 0.0.0.0:4317 }, http: { endpoint: 0.0.0.0:4318 } }
  processors:
    memory_limiter: { check_interval: 1s, limit_percentage: 80, spike_limit_percentage: 20 }
    batch: {}
  exporters:
    otlphttp/mimir:
      endpoint: http://mimir.mimir.svc:8080/otlp
      headers: { X-Scope-OrgID: homelab }
    otlphttp/loki:
      endpoint: http://loki.loki.svc:3100/otlp
      headers: { X-Scope-OrgID: homelab }
    otlp/tempo:
      endpoint: tempo.tempo.svc:4317
      tls: { insecure: true }
      headers: { X-Scope-OrgID: homelab }
  service:
    pipelines:
      metrics: { receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlphttp/mimir] }
      logs:    { receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlphttp/loki] }
      traces:  { receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlp/tempo] }
```

```bash
helm install otel-agent   open-telemetry/opentelemetry-collector -n otel -f otel-agent.yaml
helm install otel-cluster open-telemetry/opentelemetry-collector -n otel -f otel-cluster.yaml
helm install otel-gateway open-telemetry/opentelemetry-collector -n otel -f otel-gateway.yaml
```

**7d. NetworkPolicies (the Private Link stand-in).** In the `mimir`, `loki`, and `tempo` namespaces, allow ingress only from the `otel` namespace (write) and the `grafana` namespace (read), plus traffic from within the same namespace (Mimir's memberlist needs it). This needs a CNI that enforces NetworkPolicy (Calico or Cilium; k3s's default flannel setup enforces it through its built-in policy controller).

**Label discipline** still applies: keep pod names and IDs out of Loki labels; they belong in structured metadata.

## Step 8: Grafana OSS

Install Grafana from the `grafana-community` chart with data sources provisioned in values, so Grafana itself holds nothing you can't rebuild from Git.

**SQLite or PostgreSQL?** The article puts Grafana's metadata in RDS PostgreSQL so several Grafana replicas can share it. With one replica, SQLite on a small PVC is fine. Add PostgreSQL (CloudNativePG operator, or Postgres on the NAS VM) only if you want 2+ Grafana replicas.

```yaml
# grafana-values.yaml
nodeSelector: { workload: observability }
tolerations: [{ key: workload, value: observability, effect: NoSchedule }]
persistence: { enabled: true, storageClassName: local-path, size: 2Gi }
admin:
  existingSecret: grafana-admin          # create it first; don't put the password in values
resources: { requests: { cpu: 200m, memory: 256Mi }, limits: { memory: 512Mi } }

datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Mimir
        uid: mimir
        type: prometheus
        url: http://mimir.mimir.svc:8080/prometheus
        isDefault: true
        jsonData:
          httpHeaderName1: X-Scope-OrgID
          exemplarTraceIdDestinations: [{ name: trace_id, datasourceUid: tempo }]
        secureJsonData: { httpHeaderValue1: homelab }
      - name: Loki
        uid: loki
        type: loki
        url: http://loki.loki.svc:3100
        jsonData:
          httpHeaderName1: X-Scope-OrgID
          derivedFields:
            - { name: TraceID, matcherType: label, matcherRegex: trace_id, datasourceUid: tempo, url: "$${__value.raw}" }
        secureJsonData: { httpHeaderValue1: homelab }
      - name: Tempo
        uid: tempo
        type: tempo
        url: http://tempo.tempo.svc:3200
        jsonData:
          httpHeaderName1: X-Scope-OrgID
          tracesToLogsV2: { datasourceUid: loki, filterByTraceID: true }
          serviceMap: { datasourceUid: mimir }
        secureJsonData: { httpHeaderValue1: homelab }

dashboardProviders:
  dashboardproviders.yaml:
    apiVersion: 1
    providers:
      - { name: git, folder: Homelab, type: file, options: { path: /var/lib/grafana/dashboards/git } }
```

```bash
kubectl create ns grafana
kubectl -n grafana create secret generic grafana-admin --from-literal=admin-user=admin --from-literal=admin-password=<strong>
helm install grafana grafana-community/grafana -n grafana -f grafana-values.yaml
```

**Reaching it:** expose only Grafana, through whatever ingress your cluster already has (Traefik ships with k3s), at something like `grafana.lab`. Mimir, Loki, Tempo, and Garage's admin port stay internal. If you want it off-LAN, put it behind Tailscale or Cloudflare Access rather than a port forward.

**Good first dashboards:** Kubernetes cluster and node views built on kubeletstats and hostmetrics, a Loki logs explorer, and Tempo's service graph (once you enable Tempo's metrics-generator to remote-write into Mimir).

## Step 9: Monitor Proxmox, the VMs, and the stack itself

Scrape the Proxmox host, the NAS and Garage VMs, and the stack's own `/metrics` through the cluster collector, then add one check that lives outside the stack so you hear when the whole thing is down.

| Target | How | Notes |
| --- | --- | --- |
| Proxmox host and guests | `prometheus-pve-exporter` in the cluster | Proxmox API token with the read-only **PVEAuditor** role; gives per-VM CPU, RAM, disk, and up/down |
| Proxmox disk health | `smartctl_exporter` on the host | The single internal drive is your biggest risk; alert on reallocated sectors and wear |
| NAS VM, Garage VM | `node_exporter` on each | Filesystem fill is the alert that matters most |
| Garage | Built-in `/metrics` on port 3903 | Bearer token = `metrics_token` from `garage.toml` |
| Mimir, Loki, Tempo, Grafana | Their own `/metrics` | Import the Mimir, Loki, and Tempo mixin dashboards |

Add these scrape jobs to the cluster collector's `prometheus` receiver (7b) so they're scraped once, not once per node.

**Create the Proxmox token** (on the host):

```bash
pveum user add monitoring@pve
pveum aclmod / -user monitoring@pve -role PVEAuditor
pveum user token add monitoring@pve exporter --privsep 0
```

**Alerts worth having on day one:**

- Proxmox host RAM above 90% or root/data storage above 85%
- Garage data disk above 80% (Mimir, Loki, and Tempo all stop writing when it fills)
- Any node NotReady for 5 minutes
- OTel gateway export failures (`otelcol_exporter_send_failed_*`) above zero for 10 minutes
- SMART pre-fail attributes changing

**Watching the watcher.** If the stack is down, it can't alert. Two cheap options: a Grafana alert rule named `Watchdog` that always fires and pings a dead-man's-switch service (for example healthchecks.io) every few minutes, so silence means trouble; and Uptime Kuma on the NAS VM probing `grafana.lab` and Garage's health endpoint. Send notifications to something on your phone (Discord, ntfy, Pushover).

## Step 10: GitOps, backups, and single-host durability

Put every manifest and values file in Git and let Flux or Argo CD apply them, and copy Garage's buckets off the physical host, because a backup on the same drive isn't a backup.

**10a. GitOps layout**

| Path | Contents | Applied by |
| --- | --- | --- |
| `infra/proxmox/` | VM definitions (Terraform `bpg/proxmox` provider) for obs workers and `garage-1` | You, from your workstation |
| `clusters/homelab/observability/` | Mimir manifests, HelmReleases for Loki, Tempo, Grafana, collectors | Flux or Argo CD |
| `clusters/homelab/observability/dashboards/` | Dashboard JSON, loaded by Grafana's provisioning or sidecar | Flux or Argo CD |
| `clusters/homelab/secrets/` | S3 keys, Grafana admin, Proxmox token, encrypted | SOPS + age, or Sealed Secrets |

Codifying the VMs with Terraform turns your golden-image clones into something you can rebuild with one command, which pairs well with the snapshots you already keep. It's also the closest homelab practice to the Azure guide's Terraform layer.

**10b. What lives where, and how to get it back**

| Data | Lives in | Backup | Restore |
| --- | --- | --- | --- |
| Metrics blocks, log chunks, traces | Garage buckets | Nightly `rclone sync` to an off-host target | Sync back, restart pods |
| Dashboards, data sources, alert rules | Git | Git remote (GitHub) | Re-provision on install |
| Grafana users, stars, ad-hoc dashboards | Grafana SQLite PVC | Optional; or keep everything in Git | Reinstall |
| WAL and recent in-memory data | local-path PVCs | None needed | Lost hours are acceptable |
| VM definitions | Terraform + golden image | Git + Proxmox template | `terraform apply` |

**10c. Off-host copy of Garage**

```bash
# rclone remote for Garage (path-style S3)
rclone config create garage s3 provider=Other endpoint=http://garage.lab:3900 \
  access_key_id=<backup-key-id> secret_access_key=<backup-secret> force_path_style=true

# nightly, from the NAS VM or a CronJob
for b in mimir-blocks loki-chunks tempo-traces mimir-ruler loki-ruler; do
  rclone sync garage:$b offsite:homelab-obs/$b --fast-list
done
```

Create a dedicated read-only Garage key for this (`garage bucket allow --read`). For `offsite`, use an external USB disk passed through to the NAS VM, a second machine, or a cheap cloud bucket (Backblaze B2, Cloudflare R2). Copying to the NAS VM alone doesn't protect against the internal drive failing, since both sit on it.

Exclude Garage's data disk from `vzdump` jobs (**VM > Hardware > disk > Backup: off**) so Proxmox backups don't duplicate hundreds of GB; rclone covers that data.

**Be honest about the limits.** One host means a host reboot is a full observability outage, and replication factor 1 means one corrupted disk loses whatever wasn't copied off. That's fine for a homelab; the skills (object-storage-backed engines, gateways, tenancy, GitOps) transfer directly to the Azure build.

## Validation checklist and pitfalls

Run the checklist top to bottom after the first install; the pitfalls table covers what usually breaks on a small single-host build.

- [ ] `kubectl get nodes` shows both obs workers Ready, labeled, and tainted
- [ ] `garage status` shows one healthy node with the layout applied; all six buckets exist
- [ ] Mimir, Loki, Tempo, and Grafana pods are scheduled only on obs workers
- [ ] Grafana Explore returns `up` from Mimir, recent pod logs from Loki, and a trace from Tempo
- [ ] Clicking a trace ID in a log line opens it in Tempo
- [ ] Within a few hours, `garage bucket info` shows objects growing in each bucket
- [ ] Proxmox host, VM, and disk metrics appear; the Watchdog ping reaches your dead-man's switch
- [ ] A restore drill: `rclone sync` one bucket back to a scratch bucket and verify object counts

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `NoSuchBucket` or DNS errors to `mimir-blocks.garage.lab` | Client using virtual-host addressing | Set path style: `bucket_lookup_type: path` (Mimir), `s3ForcePathStyle` (Loki), `forcepathstyle` (Tempo) |
| Garage rejects every write | Layout never applied | `garage layout assign` then `garage layout apply --version 1` |
| Loki install leaves pods Pending | Default memcached caches request several GB | Disable `chunksCache` and `resultsCache` |
| Mimir won't start, mentions Kafka or ingest storage | Ingest storage on | `ingest_storage.enabled: false` and `ingester.push_grpc_method_enabled: true` |
| `${AWS_ACCESS_KEY_ID}` used literally | Env expansion off | Add `-config.expand-env=true` |
| 401 "no org id" from Mimir, Loki, or Tempo | Missing tenant header | Add `X-Scope-OrgID: homelab` at the gateway and in each Grafana data source |
| OOMKilled ingesters after adding apps | Series or stream growth | Raise memory limit, then find the high-cardinality label (`topk` by metric name) |
| Everything down after a Proxmox reboot | VMs don't autostart, or start in the wrong order | `onboot 1` and a startup order: NAS, Garage, control plane, workers |
| Two nodes fight over the same identity | Clones share a machine-id | Regenerate `/etc/machine-id` on each clone |

## Sources

- [MinIO archived: self-hosted S3 alternatives (Pinggy)](https://pinggy.io/blog/minio_archived_self_hosted_s3_alternatives/)
- [MinIO successor comparison (WZ-IT, Aug 2026)](https://wz-it.com/en/blog/minio-successor-s3-storage-comparison/)
- [Deploy Garage S3 object storage (RamNode, Garage v2.3.0)](https://www.ramnode.com/guides/garage)
- [Install the monolithic Loki Helm chart](https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic.md)
- [Loki 3.6 release notes (chart move, SSD deprecation)](https://grafana.com/docs/enterprise-logs/latest/release-notes/v3-6)
- [Configure the Grafana Mimir Kafka backend](https://grafana.com/docs/mimir/next/configure/configure-kafka-backend.md)
- [Migrate the Mimir Helm chart from 5.x to 6.0 (classic architecture)](https://grafana.com/docs/helm-charts/mimir-distributed/latest/migration-guides/migrate-helm-chart-5.x-to-6.0/)
- [Migrate from Tempo 2.x to 3.0 (monolithic without Kafka)](https://grafana.com/docs/tempo/latest/set-up-for-tracing/setup-tempo/migrate-to-3/)
- [Tempo Helm chart moved to grafana-community](https://grafana.com/docs/helm-charts/tempo-distributed/next/get-started-helm-charts/)
- [Community Grafana Helm chart (successor to grafana/helm-charts)](https://images.chainguard.dev/directory/community-chart/community-grafana/overview)
