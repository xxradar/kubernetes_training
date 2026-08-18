# Module 1.2: Helm Fundamentals

**Duration:** 1 hour
**Objective:** Understand Helm concepts and practice installing, upgrading, and rolling back releases

## What You'll Learn

- Core Helm concepts: Charts, Releases, Repositories, Values
- How to find and inspect charts
- Install, upgrade, and rollback releases
- Customize deployments with values files

## Why This Matters

Every production Kubernetes deployment uses Helm. It's the standard way to:
- Package applications for deployment
- Manage configuration across environments
- Track deployment history and rollback when needed

---

## Helm Concepts

Before we start, let's understand the terminology:

| Concept | Description | Analogy |
|---------|-------------|---------|
| **Chart** | Package of Kubernetes resources | Like an apt/yum package |
| **Release** | Running instance of a chart | Like an installed package |
| **Repository** | Collection of charts | Like apt sources |
| **Values** | Configuration for a chart | Like package config files |

---

## Step 1: Explore Helm Repositories

### 1.1 Add the Bitnami repository

Bitnami maintains high-quality charts for common applications:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

### 1.2 Update repository index

```bash
helm repo update
```

### 1.3 List configured repositories

```bash
helm repo list
```

Expected output:
```
NAME    URL
bitnami https://charts.bitnami.com/bitnami
```

### 1.4 Search for charts

```bash
# Search for nginx
helm search repo nginx

# Search with versions
helm search repo nginx --versions | head -10
```

> 💭 **Think about it:** How is `helm repo` similar to `apt` on Ubuntu? What are the advantages of having a central repository?

---

## Step 2: Inspect a Chart Before Installing

Always look before you leap!

### 2.1 View chart metadata

```bash
helm show chart bitnami/nginx
```

This shows the Chart.yaml with name, version, description.

### 2.2 View default values

```bash
helm show values bitnami/nginx | head -50
```

This is crucial — it shows all configurable options.

### 2.3 View everything

```bash
helm show all bitnami/nginx | head -100
```

---

## Step 3: Create a Practice Cluster

We'll create a temporary cluster for Helm practice:

```bash
kind create cluster --name helm-practice
```

Verify it's running:

```bash
kubectl cluster-info
kubectl get nodes
```

### ⚠️ Troubleshooting: Cluster Creation Issues

If kind cluster creation fails (e.g., cgroup v2 compatibility errors), see the troubleshooting section in [Module 1.3](../1.3-kind-cilium/README.md#troubleshooting-cgroup-v2-compatibility).

**Alternative: Use the Dry-Run Workflow**

If you cannot create a cluster, you can still learn all Helm concepts using the dry-run workflow below. This demonstrates Helm's templating and packaging without requiring a running cluster.

---

## Alternative: Dry-Run Workflow (No Cluster Required)

> 💡 **Use this section if cluster creation fails.** You'll learn the same concepts using `--dry-run` and `helm template`.

### Dry-Run Installation

See what Helm would create without actually deploying:

```bash
# Dry-run shows what would be installed
helm install my-nginx bitnami/nginx --dry-run
```

This outputs:
- Release metadata
- Computed values
- All Kubernetes manifests that would be created

### Template Rendering

Generate manifests without connecting to a cluster:

```bash
# Template renders charts to YAML
helm template my-nginx bitnami/nginx > nginx-manifests.yaml

# View the generated manifests
cat nginx-manifests.yaml | head -100
```

### Template with Custom Values

```bash
# Create a values file
cat > my-values.yaml << 'EOF'
replicaCount: 3

service:
  type: ClusterIP
  port: 8080

resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
EOF

# Template with custom values
helm template my-nginx bitnami/nginx -f my-values.yaml > nginx-custom.yaml

# Verify your values were applied
grep -A2 "replicas:" nginx-custom.yaml
grep -A5 "ports:" nginx-custom.yaml
```

### Validate Charts

```bash
# Lint checks chart for errors
helm lint bitnami/nginx

# Dry-run validates against Kubernetes API schema
helm install my-nginx bitnami/nginx --dry-run --debug 2>&1 | head -50
```

### Compare Configurations

```bash
# Generate manifests with different values and compare
helm template nginx-small bitnami/nginx --set replicaCount=1 > small.yaml
helm template nginx-large bitnami/nginx --set replicaCount=5 > large.yaml

# Compare the differences
diff small.yaml large.yaml | head -20
```

### Key Dry-Run Commands Summary

```bash
helm install <name> <chart> --dry-run     # Preview installation
helm template <name> <chart>               # Render to YAML (no cluster needed)
helm template <name> <chart> -f values.yaml # Render with custom values
helm lint <chart>                          # Validate chart syntax
```

> 💭 **Think about it:** Dry-run and template commands are invaluable in CI/CD pipelines. How would you use them to validate changes before deploying to production?

**Once you have a working cluster**, continue with Step 4 below to practice actual installations.

---

## Step 4: Install Your First Chart

### 4.1 Install nginx with defaults

```bash
helm install my-nginx bitnami/nginx
```

**Anatomy of the command:**
- `helm install` — action
- `my-nginx` — release name (you choose this)
- `bitnami/nginx` — chart to install

### 4.2 Check the release

```bash
# List releases
helm list

# Get detailed status
helm status my-nginx
```

### 4.3 See what was created

```bash
kubectl get all
```

You should see:
- Deployment
- ReplicaSet
- Pod(s)
- Service

> 💭 **Think about it:** We ran one command, but Kubernetes created multiple resources. What's the benefit of this abstraction?

---

## Step 5: Customize with Values

### 5.1 Install with inline values

```bash
helm install my-nginx2 bitnami/nginx --set replicaCount=3
```

Check the pods:

```bash
kubectl get pods
```

You should see 3 nginx pods.

### 5.2 Install with a values file

Create a values file:

```bash
cat > my-values.yaml << 'EOF'
replicaCount: 2

service:
  type: ClusterIP
  port: 8080

resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "100m"
EOF
```

Install using the file:

```bash
helm install my-nginx3 bitnami/nginx -f my-values.yaml
```

Verify the configuration was applied:

```bash
kubectl get svc my-nginx3
kubectl get deployment my-nginx3 -o jsonpath='{.spec.replicas}'
echo ""  # newline
```

---

## Step 6: Upgrade a Release

### 6.1 Change the replica count

```bash
helm upgrade my-nginx bitnami/nginx --set replicaCount=5
```

Watch the pods scale up:

```bash
kubectl get pods -w
# Press Ctrl+C to exit
```

### 6.2 Check the release history

```bash
helm history my-nginx
```

You should see two revisions.

---

## Step 7: Rollback a Release

### 7.1 Roll back to the previous version

```bash
helm rollback my-nginx 1
```

### 7.2 Verify the rollback

```bash
kubectl get pods
helm history my-nginx
```

The pods should be back to the original count, and history shows revision 3 (rollback).

> 💭 **Think about it:** Why is rollback capability important in production? What would you do without Helm?

---

## Step 8: Get Release Information

### 8.1 Get the values used for a release

```bash
# User-supplied values only
helm get values my-nginx

# All values (including defaults)
helm get values my-nginx --all
```

### 8.2 Get the rendered manifests

```bash
helm get manifest my-nginx | head -50
```

This shows exactly what Kubernetes resources were created.

---

## Step 9: Uninstall Releases

```bash
# Uninstall all our test releases
helm uninstall my-nginx
helm uninstall my-nginx2
helm uninstall my-nginx3

# Verify
helm list
kubectl get all
```

---

## Step 10: Clean Up

Delete the practice cluster:

```bash
kind delete cluster --name helm-practice
```

---

## Lab Complete! ✅

You now understand Helm fundamentals.

**What you learned:**
- Adding and searching repositories
- Inspecting charts before installing
- Installing with defaults and custom values
- Upgrading and rolling back releases
- Getting release information
- Using `--dry-run` and `helm template` for validation (CI/CD patterns)

**Key commands:**
```bash
helm repo add/update/list     # Manage repositories
helm search repo              # Find charts
helm show chart/values/all    # Inspect charts
helm install                  # Deploy a chart
helm upgrade                  # Update a release
helm rollback                 # Revert to previous
helm list                     # Show releases
helm history                  # Release history
helm get values/manifest      # Release details
helm uninstall                # Remove release
```

**Next:** [Module 1.3 - kind + Cilium CNI](../1.3-kind-cilium/)

---

## 💭 Thinking Questions

1. What happens to your data if you `helm uninstall` an application with a database?
2. How would you handle secrets (like passwords) in a values file securely?
3. Why might you use `--set` vs a values file? When would you choose each?
4. If a Helm upgrade fails halfway through, what state are your resources in?

---

## Quick Reference Card

```bash
# Install
helm install <name> <chart>
helm install <name> <chart> -f values.yaml
helm install <name> <chart> --set key=value
helm install <name> <chart> --namespace <ns> --create-namespace

# Inspect
helm show chart <chart>
helm show values <chart>

# Manage
helm list [-A]
helm status <release>
helm upgrade <release> <chart>
helm rollback <release> <revision>
helm uninstall <release>

# Debug
helm get values <release>
helm get manifest <release>
helm history <release>

# Dry-Run / Template (no cluster required)
helm install <name> <chart> --dry-run      # Preview what would be installed
helm template <name> <chart>                # Render manifests to YAML
helm template <name> <chart> -f values.yaml # Render with custom values
helm lint <chart>                           # Validate chart syntax
```
