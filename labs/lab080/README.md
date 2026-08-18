# LAB080 - Introduction to Helm

**Helm** is the package manager for Kubernetes. Instead of applying a pile of individual YAML manifests, you install a **chart**: a versioned, parameterised package that expands into all the resources an app needs (Deployment, Service, ConfigMap, and so on). It is to Kubernetes what `apt` or `yum` is to a Linux distro.

Four terms cover most of it:

| Concept | What it is | Analogy |
|---|---|---|
| **Chart** | A package of templated Kubernetes manifests | an apt/yum package |
| **Release** | One installed instance of a chart | an installed package |
| **Repository** | A collection of charts you can pull from | apt sources |
| **Values** | The configuration you feed a chart | a package's config file |

> Runs on the kind cluster from LAB000. Requires the `helm` CLI.

> **Chart source note.** Many older tutorials use Bitnami charts. The chart *source* is still open (Apache-2.0 on GitHub), but the distribution changed in 2025: the `helm repo add https://charts.bitnami.com/bitnami` method is deprecated in favour of OCI (`oci://registry-1.docker.io/bitnamicharts/<chart>`), the images moved to hardened *Bitnami Secure Images*, and the previous free Debian images were relegated to an unsupported `bitnamilegacy` registry (full access needs a Broadcom subscription). Older `bitnami/nginx`-style tutorials can therefore fail on image pulls. This lab uses **podinfo**, a small, free, self-contained chart that is ideal for practising install / upgrade / rollback.

## 1. Add a repository and search it
```
helm repo add podinfo https://stefanprodan.github.io/podinfo
helm repo update
helm repo list
```
Search the repo (and list versions):
```
helm search repo podinfo
helm search repo podinfo --versions | head
```

## 2. Inspect a chart before installing it
Always look before you install. Read the chart metadata and its configurable values:
```
helm show chart podinfo/podinfo
helm show values podinfo/podinfo | head -40
```
`helm show values` is the important one: it lists every knob you can override.

### Pull and unpack the chart to see the files
To see what a chart is actually made of, pull and untar it:
```
helm pull podinfo/podinfo --untar
ls podinfo
```
You get the chart's directory layout:
```
podinfo/
├── Chart.yaml        # name, version, description
├── values.yaml       # default values (the knobs)
├── templates/        # the templated manifests (Deployment, Service, HPA, ...)
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ...
└── charts/           # bundled sub-charts (dependencies), if any
```
Look at a template and the defaults it references:
```
sed -n '1,30p' podinfo/templates/deployment.yaml
sed -n '1,20p' podinfo/values.yaml
```
This is the whole idea of Helm: `templates/` + `values.yaml` render into final manifests.

## 3. Render locally, without a cluster
`helm template` expands a chart to plain YAML on your machine, no cluster or install required. This is what you use in CI to review or validate what a change would produce:
```
helm template demo podinfo/podinfo | head -60
```
`--dry-run` does the same through the API server (so it also validates against the cluster's schema):
```
helm install demo podinfo/podinfo --dry-run | head -40
```
Render with an override and confirm it took effect:
```
helm template demo podinfo/podinfo --set replicaCount=3 | grep -i replicas
```

## 4. Install a chart
```
helm install my-podinfo podinfo/podinfo
```
Anatomy: `helm install` (action) `my-podinfo` (the release name you choose) `podinfo/podinfo` (the chart). Check the release and what it created:
```
helm list
helm status my-podinfo
kubectl get all -l app.kubernetes.io/name=podinfo
```
One command produced a Deployment, ReplicaSet, Pod(s), and a Service.

## 5. Customize with values
Inline with `--set`:
```
helm upgrade my-podinfo podinfo/podinfo --set replicaCount=3
kubectl get pods -l app.kubernetes.io/name=podinfo
```
Or with a values file, the reproducible way:
```
cat > my-values.yaml <<EOF
replicaCount: 2
ui:
  message: "Hello from LAB080"
resources:
  requests:
    cpu: 50m
    memory: 32Mi
EOF
helm upgrade my-podinfo podinfo/podinfo -f my-values.yaml
```
Verify the applied config:
```
kubectl get deploy my-podinfo -o jsonpath='{.spec.replicas}{"\n"}'
helm get values my-podinfo
```

## 6. Upgrade and history
Every install/upgrade is a numbered **revision**:
```
helm upgrade my-podinfo podinfo/podinfo --set replicaCount=4
helm history my-podinfo
```

## 7. Rollback
Roll back to an earlier revision (here revision 1):
```
helm rollback my-podinfo 1
helm history my-podinfo
kubectl get pods -l app.kubernetes.io/name=podinfo
```
**Expected:** the pod count returns to the revision-1 state, and `helm history` records the rollback as a new revision. This is why teams package with Helm: one command reverts a bad deploy.

## 8. Inspect a release
```
helm get values my-podinfo          # user-supplied values
helm get values my-podinfo --all    # values including defaults
helm get manifest my-podinfo | head # exactly what was applied to the cluster
```

## 9. Uninstall
```
helm uninstall my-podinfo
helm list
```

## Explore it yourself
* Change `ui.message` or `ui.color` in `my-values.yaml`, upgrade, and port-forward to the service (`kubectl port-forward svc/my-podinfo 9898:9898`) to see it.
* Run `helm template` before an upgrade and `diff` it against `helm get manifest` of the running release. What changed?
* Add `--namespace demo --create-namespace` to an install. Where do the resources land, and how does `helm list -A` differ from `helm list`?
* How would you keep a database's data across a `helm uninstall`? (Hint: PVCs and reclaim policy from LAB061.)

## Quick reference
```
helm repo add/update/list                  # manage repositories
helm search repo <term> [--versions]       # find charts
helm show chart|values|all <chart>         # inspect a chart
helm pull <chart> --untar                  # download + unpack to see files
helm template <name> <chart> [-f values]   # render to YAML (no cluster)
helm install <name> <chart> [-f|--set]     # deploy a release
helm upgrade <name> <chart> [-f|--set]     # update a release
helm rollback <name> <revision>            # revert
helm list [-A]                             # releases
helm history <name>                        # revisions
helm get values|manifest <name>            # release details
helm uninstall <name>                      # remove
```

> Takeaway: Helm packages an app's manifests into a versioned **chart**; you install it as a **release** and shape it with **values**. `templates/` plus `values.yaml` render into the final YAML, which you can preview with `helm template`/`--dry-run` before it ever touches the cluster. Every install and upgrade is a numbered revision, so `helm rollback` makes reverting a change a single command.
