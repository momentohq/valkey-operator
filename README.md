# Momento Valkey Operator

A Kubernetes operator for managing [Valkey](https://valkey.io/) clusters. It handles cluster bootstrapping, topology changes, rolling image upgrades, and ACL management — all driven by Kubernetes custom resources.

## How it works

The operator introduces four custom resource types:

| Resource | Scope | Purpose |
|---|---|---|
| `ValkeyImage` | Cluster | Allowlist of approved Valkey container images |
| `ValkeyConfig` | Cluster | Named configuration bundles (image, resources, Valkey settings) |
| `ValkeyRole` | Cluster | Reusable command permission sets for ACLs |
| `ValkeyCluster` | Namespace | Provisions a cluster by referencing a config and specifying topology |

The typical workflow is: define the images and configs you want to offer, then let teams provision clusters by creating `ValkeyCluster` resources in their namespaces.

---

## Prerequisites

- Kubernetes 1.26+
- `kubectl` configured against your cluster

---

## Install

### 1. Apply CRDs

```bash
kubectl apply -f https://github.com/momentohq/valkey-operator/releases/download/v0.6.0/crds.json
```

### 2. Deploy the operator

```bash
kubectl apply -f https://github.com/momentohq/valkey-operator/releases/download/v0.6.0/operator.yaml
```

> The operator image is pulled from Docker Hub at `gomomento/valkey-operator`.

Verify the operator is running:

```bash
kubectl -n valkey-operator rollout status deployment/valkey-operator
```

---

## Quick start

### 1. Register an image

`ValkeyImage` is the allowlist of container images clusters can use. Create one for the Valkey version you want to run:

```bash
kubectl apply -f - <<'EOF'
apiVersion: valkey.gomomento.com/v1alpha1
kind: ValkeyImage
metadata:
  name: valkey-9-0
spec:
  repository: valkey/valkey
  tag: "9.0.0"
  version: "9.0.0"
EOF
```

### 2. Create a configuration

`ValkeyConfig` bundles an image reference, resource limits, and Valkey settings into a named profile. Clusters reference this profile rather than specifying settings directly.

```bash
kubectl apply -f - <<'EOF'
apiVersion: valkey.gomomento.com/v1alpha1
kind: ValkeyConfig
metadata:
  name: standard
spec:
  imageRef: valkey-9-0
  resources:
    cpu: "1"
    memory: "2Gi"
  valkey:
    maxmemory: "1500mb"
    maxmemory-policy: "allkeys-lru"
EOF
```

### 3. Provision a cluster

Create a `ValkeyCluster` in the namespace where your application lives:

```bash
kubectl apply -f - <<'EOF'
apiVersion: valkey.gomomento.com/v1alpha1
kind: ValkeyCluster
metadata:
  name: my-cluster
  namespace: my-app
spec:
  configRef: standard
  shards: 3
  replicasPerShard: 1
EOF
```

Watch the cluster come up:

```bash
kubectl -n my-app get valkeycluster my-cluster -w
```

Once `STATE` shows `Active`, the cluster is ready.

---

## Common operations

### Scale shards

Increase or decrease the number of shards. The operator rebalances slot assignments automatically:

```bash
kubectl -n my-app patch valkeycluster my-cluster \
  --type merge -p '{"spec": {"shards": 6}}'
```

### Scale replicas

Add or remove replicas per shard:

```bash
kubectl -n my-app patch valkeycluster my-cluster \
  --type merge -p '{"spec": {"replicasPerShard": 2}}'
```

### Switch config

Move a cluster to a different `ValkeyConfig` — for example to change its resource profile:

```bash
kubectl -n my-app patch valkeycluster my-cluster \
  --type merge -p '{"spec": {"configRef": "large"}}'
```

The operator will perform a rolling upgrade to apply the new config.

### Upgrade image

Register a new `ValkeyImage` for the new version and point the config to it. The operator performs a rolling upgrade across every cluster that references the config:

```bash
# Register the new image version
kubectl apply -f - <<'EOF'
apiVersion: valkey.gomomento.com/v1alpha1
kind: ValkeyImage
metadata:
  name: valkey-9-0-1
spec:
  repository: valkey/valkey
  tag: "9.0.1"
  version: "9.0.1"
EOF

# Update the config to point to the new image
kubectl patch valkeyconfig standard \
  --type merge -p '{"spec": {"imageRef": "valkey-9-0-1"}}'
```

All clusters using `standard` config will begin a rolling upgrade immediately.

### Config inheritance

Use `baseRef` to share common settings across multiple configs without repeating them:

```bash
kubectl apply -f - <<'EOF'
apiVersion: valkey.gomomento.com/v1alpha1
kind: ValkeyConfig
metadata:
  name: base
spec:
  imageRef: valkey-9-0
  valkey:
    maxmemory-policy: "allkeys-lru"
---
apiVersion: valkey.gomomento.com/v1alpha1
kind: ValkeyConfig
metadata:
  name: small
spec:
  baseRef: base
  resources:
    cpu: "0.5"
    memory: "512Mi"
  valkey:
    maxmemory: "400mb"
---
apiVersion: valkey.gomomento.com/v1alpha1
kind: ValkeyConfig
metadata:
  name: large
spec:
  baseRef: base
  resources:
    cpu: "4"
    memory: "16Gi"
  valkey:
    maxmemory: "14000mb"
EOF
```

### ACL management

Use `ValkeyRole` to define reusable command permission sets, then bind users to them on a `ValkeyCluster`.

**Create a role:**

```bash
kubectl apply -f - <<'EOF'
apiVersion: valkey.gomomento.com/v1alpha1
kind: ValkeyRole
metadata:
  name: read-write
spec:
  categories:
    - name: read
      access: allow
    - name: write
      access: allow
EOF
```

**Bind a user to the cluster** (password must be provided as a SHA-256 hex hash):

```bash
kubectl -n my-app patch valkeycluster my-cluster --type merge -p '{
  "spec": {
    "acl": [
      {
        "username": "app-user",
        "passwordHashes": ["<sha256-hex-of-password>"],
        "permissions": [
          {
            "roleRef": "read-write",
            "keys": [{"pattern": "myapp:*", "access": "readwrite"}]
          }
        ]
      }
    ]
  }
}'
```

ACL changes propagate to all cluster nodes without a restart.

**Rotate a password** — add the new hash alongside the old one, update your clients, then remove the old hash:

```bash
kubectl -n my-app patch valkeycluster my-cluster --type merge -p '{
  "spec": {
    "acl": [
      {
        "username": "app-user",
        "passwordHashes": ["<old-hash>", "<new-hash>"],
        "permissions": [
          {
            "roleRef": "read-write",
            "keys": [{"pattern": "myapp:*", "access": "readwrite"}]
          }
        ]
      }
    ]
  }
}'
```

### Enable TLS

When TLS is enabled the cluster runs TLS-only: client traffic, replication, and the cluster bus are all encrypted. The plain port is closed.

The operator does not manage certificates. Create a `kubernetes.io/tls` Secret in the cluster's namespace containing `tls.crt`, `tls.key`, and `ca.crt`. The certificate must include SANs covering `{cluster}.{namespace}.svc.cluster.local` and `*.{cluster}.{namespace}.svc.cluster.local`.

Reference the Secret in `spec.tls.secretRef`:

```bash
kubectl apply -f - <<'EOF'
apiVersion: valkey.gomomento.com/v1alpha1
kind: ValkeyCluster
metadata:
  name: my-cluster
  namespace: my-app
spec:
  configRef: standard
  shards: 3
  replicasPerShard: 1
  tls:
    secretRef: my-cluster-tls
EOF
```

Connect once the cluster is `Active` (from inside the cluster):

```bash
valkey-cli -h my-cluster.my-app.svc.cluster.local --tls --cacert ca.crt -p 6379 PING
```

**Cert rotation** — update the Secret's data; the operator reloads the new cert on all pods without a restart. For CA rotation, include both old and new CA PEM blocks in `ca.crt` during the transition, then remove the old CA once rotation is complete.

> TLS cannot be enabled/disabled on an existing cluster, it must be set at creation time.

### Provision certificates with cert-manager

The operator consumes a `kubernetes.io/tls` Secret but does not create or renew it. In production the recommended way to produce and rotate that Secret is [cert-manager](https://cert-manager.io). cert-manager issues the certificate, writes it into a Secret in the right shape, and renews it automatically before expiry — and the operator reloads each renewal across the cluster without restarting pods.

This requires cert-manager to be installed in the cluster (`kubectl get pods -n cert-manager` should show it running).

**1. An issuer to sign the certificate.** If you already run a cert-manager `Issuer`/`ClusterIssuer` backed by your own CA or HashiCorp Vault, use it and skip to step 2. Otherwise create a self-signed CA dedicated to this cluster — a one-time bootstrap `SelfSigned` Issuer signs a long-lived CA, and a `CA` Issuer then signs the cluster's leaf certificates from it:

```bash
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-bootstrap
  namespace: my-app
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: valkey-ca
  namespace: my-app
spec:
  isCA: true
  commonName: my-app-valkey-ca
  secretName: valkey-ca-keypair
  duration: 87600h   # 10y — keep the CA long-lived
  renewBefore: 720h
  privateKey:
    algorithm: ECDSA
    size: 256
  issuerRef:
    name: selfsigned-bootstrap
    kind: Issuer
    group: cert-manager.io
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: valkey-ca-issuer
  namespace: my-app
spec:
  ca:
    secretName: valkey-ca-keypair
EOF
```

> Use a private-CA issuer (`CA`, `SelfSigned`, or Vault) — not a public ACME issuer like Let's Encrypt. Two reasons: the certificate's SANs are internal `*.svc.cluster.local` names, and ACME can only issue for public domains you can prove control of via an HTTP/DNS challenge — there is nothing for it to validate in-cluster. A private CA issuer also populates `ca.crt` in the Secret, which the cluster needs to verify peer certificates on replication and the cluster bus; ACME issuers do not. Internal workload certificates like these are precisely the private-CA use case — the same pattern service meshes use for in-cluster mTLS.

**2. A `Certificate` for the cluster.** Its `secretName` is the Secret the `ValkeyCluster` will reference. Two requirements are easy to miss:

- **`dnsNames`** must list the cluster's two service SANs exactly — substitute your cluster name and namespace.
- **`usages`** must include both `server auth` and `client auth`. Each node serves TLS to clients *and* opens TLS connections to its peers for replication and the cluster bus, so it acts as a TLS client too — a server-auth-only certificate causes peer connections to fail.

```bash
kubectl apply -f - <<'EOF'
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-cluster-tls
  namespace: my-app
spec:
  secretName: my-cluster-tls
  duration: 2160h     # 90d
  renewBefore: 360h   # 15d
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Always
  usages:
    - server auth
    - client auth
  dnsNames:
    - my-cluster.my-app.svc.cluster.local
    - "*.my-cluster.my-app.svc.cluster.local"
  issuerRef:
    name: valkey-ca-issuer
    kind: Issuer
    group: cert-manager.io
EOF
```

**3. Point the cluster at the Secret.** Set `spec.tls.secretRef` to the same name as `secretName` above (see [Enable TLS](#enable-tls)):

```yaml
spec:
  tls:
    secretRef: my-cluster-tls
```

**Renewal is automatic.** cert-manager renews the leaf before expiry (per `renewBefore`) and rewrites the same Secret in place; the operator reloads the new certificate on every node with no restart and no dropped connections. Because leaf renewal keeps the same CA, no extra coordination is needed — keep the CA itself long-lived so it does not rotate underneath the running cluster. To rotate the CA, follow the bundle procedure under [Enable TLS](#enable-tls).

### Zone-aware placement

Pin a cluster to specific availability zones, and control how strictly each shard's nodes are spread across them:

```bash
kubectl apply -f - <<'EOF'
apiVersion: valkey.gomomento.com/v1alpha1
kind: ValkeyCluster
metadata:
  name: my-cluster
  namespace: my-app
spec:
  configRef: standard
  shards: 3
  replicasPerShard: 1
  placement:
    zones:
      - us-east-1a
      - us-east-1b
      - us-east-1c
    zoneSpread: required
    nodeSelector:
      node-pool: cache
EOF
```

- **`zones`** — restricts the cluster's pods to these availability zones. The operator enforces this with a required `nodeAffinity` on the `topology.kubernetes.io/zone` node label.
- **`nodeSelector`** — a map of node labels passed through verbatim to the pods' `nodeSelector`. Use it to pin the cluster to a specific node pool. It composes with `zones` — Kubernetes ANDs `nodeSelector` with `nodeAffinity`, so a pod must satisfy both the selector labels and (if set) the zone restriction.
- **`zoneSpread`** — controls how strictly each shard's nodes are spread across zones. The operator attaches a per-shard zone `topologySpreadConstraint` (`maxSkew: 1` over `topology.kubernetes.io/zone`) to every pod; `zoneSpread` selects its `whenUnsatisfiable` mode:
  - **`bestEffort`** — `whenUnsatisfiable: ScheduleAnyway`. Spreading is a soft preference: the scheduler favors it when scoring nodes, but will colocate a shard's nodes in one zone when other factors (capacity, image locality) outweigh it.
  - **`required`** — `whenUnsatisfiable: DoNotSchedule`. The scheduler will not place a pod where it would put two of a shard's nodes in the same zone. If no node satisfies the constraint, the pod stays `Pending` until one does — for example, until the cluster autoscaler provisions a node in the needed zone. This guarantees a single-zone outage never takes out a whole shard, at the cost of pods waiting on capacity rather than colocating.

  In `required` mode the operator additionally attaches a **cluster-wide** zone spread constraint, which keeps the *whole cluster* balanced across zones — per-shard spread alone doesn't ensure that, since every shard could split across the same two zones and leave a third empty. This constraint is always soft (`ScheduleAnyway`): Kubernetes permits only one hard zone spread constraint per pod, and the per-shard one holds it. So in `required` mode per-shard spread is guaranteed and cluster-wide balance is strongly preferred but not guaranteed.

`zoneSpread` affects only zone spreading. The operator separately spreads each shard's pods across hosts (a `topologySpreadConstraint` over `kubernetes.io/hostname`); that one is always best-effort and `zoneSpread` does not change it.

`zoneSpread` is optional. When omitted, it inherits the operator-wide default — `bestEffort`, unless changed in [operator configuration](#operator-configuration).

#### Changing placement on a running cluster

`zones` and `nodeSelector` are **hard constraints** — they pin pods to nodes, so the only way to honor a change is to move the pods. Editing either triggers a **rolling replacement**: the operator replaces every pod, one shard at a time, with a pod that satisfies the new constraints. It always brings up a replacement and (for primaries) fails over to it *before* retiring the old pod, so the rollout is non-disruptive to availability — but it does incur a failover per shard, like an image upgrade.

> **Heads up:** if the new placement is unsatisfiable (e.g. a `nodeSelector` no node matches, or a zone with no capacity), the replacement pod stays `Pending` and the rollout stalls on that shard. Existing pods are left running and serving — no data is lost — and the rollout resumes on its own once a matching node is available (for example, after the cluster autoscaler provisions one).

`zoneSpread`, by contrast, is a soft scheduler preference, so changing it does **not** replace existing pods — the new mode applies only to pods created afterward (during a later scale-up, upgrade, or replacement).

### Pod annotations

`spec.podAnnotations` is a map of annotations the operator applies verbatim to every Pod it creates for the cluster. Use it for tooling that reads pod annotations — metrics scrape configuration, service-mesh sidecar injection, cost-allocation tags, and the like.

```yaml
apiVersion: valkey.gomomento.com/v1alpha1
kind: ValkeyCluster
metadata:
  name: my-cluster
spec:
  configRef: my-config
  shards: 3
  replicasPerShard: 1
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "6379"
```

- The annotations land on the Pods only, not on the operator's other resources (Services, ConfigMaps, Secrets).
- Keys under the operator's own `valkey.gomomento.com/` prefix are reserved; the CRD rejects them at apply time.
- Editing `podAnnotations` is reconciled onto existing pods in place — the operator patches the new and changed keys onto live pods without recreating them, so changes propagate without a rolling restart. Annotations set on the pods by other controllers are left untouched.
- **Removing** a key from `podAnnotations` does not strip it from running pods — on a live pod the operator can't tell its own annotation from one another controller added. Removals take effect when a pod is next replaced (for example, during an image upgrade or node replacement).

---

## Operator configuration

The operator reads optional process-level settings from a `valkey-operator-config` ConfigMap that is included in `operator.yaml`. Each setting is optional: an unset value falls back to a built-in default, and an invalid value stops the operator at startup rather than letting it run with surprising behavior.

| ConfigMap key | Default | Effect |
|---|---|---|
| `defaultZoneSpread` | `bestEffort` | Default value for `placement.zoneSpread` on clusters that don't set it themselves — one of `bestEffort` or `required`. See [Zone-aware placement](#zone-aware-placement). |

Settings are read once at startup, so changing one means editing the ConfigMap and restarting the operator:

```bash
kubectl -n valkey-operator patch configmap valkey-operator-config \
  --type merge -p '{"data": {"defaultZoneSpread": "required"}}'
kubectl -n valkey-operator rollout restart deployment/valkey-operator
```

A `placement.zoneSpread` set on an individual `ValkeyCluster` always overrides this default.

---

## Check cluster status

```bash
# Overall state (Creating / Active / Updating)
kubectl -n my-app get valkeycluster

# Full status including pod list
kubectl -n my-app describe valkeycluster my-cluster

# Operator logs
kubectl -n valkey-operator logs deployment/valkey-operator
```

---

## Upgrade the operator

Apply the updated CRDs and operator manifest:

```bash
kubectl apply -f https://github.com/momentohq/valkey-operator/releases/download/v0.6.0/crds.json
kubectl apply -f https://github.com/momentohq/valkey-operator/releases/download/v0.6.0/operator.yaml
kubectl -n valkey-operator rollout status deployment/valkey-operator
```

---

## Uninstall

> These commands reference `v0.6.0`. If you installed a different version, use that one instead — check the deployed tag with `kubectl -n valkey-operator get deployment valkey-operator -o jsonpath='{.spec.template.spec.containers[0].image}'`.

```bash
# Remove all clusters across all namespaces (this deletes the Valkey pods)
kubectl delete valkeycluster --all -A

# Remove the operator
kubectl delete -f https://github.com/momentohq/valkey-operator/releases/download/v0.6.0/operator.yaml

# Remove CRDs — also deletes any remaining custom resources
kubectl delete -f https://github.com/momentohq/valkey-operator/releases/download/v0.6.0/crds.json
```

---

## Links

- [GitHub repository](https://github.com/momentohq/valkey-operator)
- [Releases & changelog](https://github.com/momentohq/valkey-operator/releases)
