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

Replace `v1.0.0` with the latest release from the [releases page](https://github.com/momentohq/valkey-operator/releases).

### 1. Apply CRDs

```bash
kubectl apply -f https://github.com/momentohq/valkey-operator/releases/download/v1.0.0/crds.json
```

### 2. Deploy the operator

```bash
kubectl apply -f https://github.com/momentohq/valkey-operator/releases/download/v1.0.0/operator.yaml
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

### Switch a cluster to a different config

Move a cluster to a different `ValkeyConfig` — for example to change its resource profile:

```bash
kubectl -n my-app patch valkeycluster my-cluster \
  --type merge -p '{"spec": {"configRef": "large"}}'
```

The operator will perform a rolling upgrade to apply the new config.

### Rolling image upgrade

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

### Zone-aware placement

Pin a cluster to specific availability zones so each shard's nodes are spread across them:

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
EOF
```

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

Apply the updated CRDs and operator manifest for the new version:

```bash
# Replace v1.1.0 with your target version
kubectl apply -f https://github.com/momentohq/valkey-operator/releases/download/v1.1.0/crds.json
kubectl apply -f https://github.com/momentohq/valkey-operator/releases/download/v1.1.0/operator.yaml
kubectl -n valkey-operator rollout status deployment/valkey-operator
```

---

## Uninstall

```bash
# Remove all clusters across all namespaces (this deletes the Valkey pods)
kubectl delete valkeycluster --all -A

# Remove the operator (replace v1.0.0 with your installed version)
kubectl delete -f https://github.com/momentohq/valkey-operator/releases/download/v1.0.0/operator.yaml

# Remove CRDs — also deletes any remaining custom resources (replace v1.0.0 with your installed version)
kubectl delete -f https://github.com/momentohq/valkey-operator/releases/download/v1.0.0/crds.json
```

---

## Links

- [GitHub repository](https://github.com/momentohq/valkey-operator)
- [Releases & changelog](https://github.com/momentohq/valkey-operator/releases)
