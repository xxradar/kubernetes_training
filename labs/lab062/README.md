# LAB062 - ConfigMaps and Secrets

An app needs two things besides its code: **configuration** (URLs, flags, tuning) and **credentials** (passwords, tokens, keys). Kubernetes stores these as small API objects and injects them into pods, either as **files** or as **environment variables**. A **ConfigMap** holds non-sensitive config; a **Secret** holds sensitive data.

Unlike the volumes in LAB061, these are **not provisioned storage**: there is no StorageClass, no PV. They are objects that live in **etcd**, are capped around ~1 MiB, and are **namespaced** (must be in the same namespace as the pod that uses them).

> Standalone lab. Namespace `config-demo`. Runs on the kind cluster from LAB000.

```
kubectl create ns config-demo
```

## A. ConfigMap
Create one from literal values (you can also use `--from-file`):
```
kubectl create configmap app-config -n config-demo \
  --from-literal=APP_COLOR=blue \
  --from-literal=APP_TIER=frontend
kubectl get configmap app-config -n config-demo -o yaml
```

### Consume it as environment variables
`envFrom` injects every key as an env var; use `valueFrom` for a single key:
```
kubectl apply -n config-demo -f pod-config-env.yaml
kubectl wait --for=condition=Ready pod/config-env -n config-demo --timeout=60s
kubectl exec -n config-demo config-env -- printenv APP_COLOR APP_TIER
```

### Consume it as files (a volume)
Mounted as a volume, **each key becomes a file** whose contents are the value. From inside the container it looks like just another directory:
```
kubectl apply -n config-demo -f pod-config-volume.yaml
kubectl wait --for=condition=Ready pod/config-vol -n config-demo --timeout=60s
kubectl exec -n config-demo config-vol -- sh -c 'ls /etc/appconfig; echo; cat /etc/appconfig/APP_COLOR'
```

## B. Secret
A Secret is the same idea, for sensitive data. Create a generic one:
```
kubectl create secret generic db-cred -n config-demo \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=s3cr3t
```
Now the single most important point for a security engineer, **base64 is not encryption**:
```
kubectl get secret db-cred -n config-demo -o jsonpath='{.data.DB_PASSWORD}'; echo
# -> czNjcjN0   (just base64)
kubectl get secret db-cred -n config-demo -o jsonpath='{.data.DB_PASSWORD}' | base64 -d; echo
# -> s3cr3t     (anyone who can read the Secret sees the value)
```
What actually protects a Secret is **RBAC** (who is allowed to read it) plus optional **encryption-at-rest** for etcd (EncryptionConfiguration / KMS). The base64 is only encoding.

### Consume it (prefer files over env)
Env vars are convenient but leakier (they show up in logs, crash dumps, `/proc/<pid>/environ`, and child processes). Consume the Secret as env vars:
```
kubectl apply -n config-demo -f pod-secret-env.yaml
kubectl wait --for=condition=Ready pod/secret-env -n config-demo --timeout=60s
kubectl exec -n config-demo secret-env -- printenv DB_USER DB_PASSWORD
```
Mounted as a volume instead, the values land as files in **tmpfs (RAM)** on the node, never written to the node's disk:
```
kubectl apply -n config-demo -f pod-secret-volume.yaml
kubectl wait --for=condition=Ready pod/secret-vol -n config-demo --timeout=60s
kubectl exec -n config-demo secret-vol -- sh -c 'cat /etc/secrets/DB_PASSWORD; echo'
```
For secrets, prefer the **file mount** over env when you can.

## C. The update gotcha
Change a ConfigMap or Secret and the behaviour differs by how it was consumed:

- **Volume-mounted** keys update **automatically** in the running pod (after a short sync delay, up to ~1 minute).
- **Environment variables** and **`subPath`** mounts do **not** update, they are frozen at pod start and only refresh on a pod restart.

See it:
```
kubectl edit configmap app-config -n config-demo    # change APP_COLOR to green
```
wait ~60s, then:
```
kubectl exec -n config-demo config-vol -- cat /etc/appconfig/APP_COLOR   # -> green (updated)
kubectl exec -n config-demo config-env -- printenv APP_COLOR             # -> still blue (frozen)
```
This trips people up constantly: apps that read config from **files** pick up changes live; apps that read **env vars** need a rollout (`kubectl rollout restart`).

## D. When the source of truth is a vault (pointer)
For real credential management you usually do not want the value sitting in etcd at all. Two common patterns pull secrets from an external vault (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager):

- **Secrets Store CSI Driver** - mounts the secret straight into the pod as tmpfs files at pod start, using the pod's own identity; nothing is stored in etcd (unless you enable sync).
- **External Secrets Operator (ESO)** - a controller that pulls from the vault and materialises a **native Kubernetes Secret**, which pods then consume the normal way.

Both are out of scope for this kind cluster (no vault), but the mental split is: native Secret = value lives in **etcd**; CSI / ESO = value lives in an **external vault**, fetched at runtime.

## Explore it yourself
* `kubectl describe pod secret-env -n config-demo` - are the Secret values shown? (No, but the env var *names* are.)
* Recreate `app-config` with `--from-file=` pointing at a real config file. How is a whole-file key mounted?
* Mark a ConfigMap `immutable: true`. What can you no longer do, and why is that good for a busy control plane?
* Which of your app's settings belong in a ConfigMap vs a Secret? (Rule of thumb: would it hurt to print it in a log?)

## Cleanup
```
kubectl delete ns config-demo
```

> Takeaway: ConfigMaps and Secrets are small, namespaced, etcd-backed API objects injected into pods as **files or env vars**. A Secret adds only base64 encoding, **not encryption**, so RBAC and encryption-at-rest are the real controls, tmpfs and file mounts beat env vars for credentials, and only **volume-mounted** values update live (env and `subPath` are frozen until restart). For vault-backed secrets, reach for the Secrets Store CSI Driver or External Secrets Operator.
