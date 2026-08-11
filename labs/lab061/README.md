# LAB061 - Intro to storage (volumes, hostPath, PVC / PV, StorageClass)

A pod is **ephemeral**: delete it and anything written inside its container is gone. To keep data you attach a **volume**. Kubernetes storage boils down to two questions: **where does the data physically live**, and **how is that storage handed to the pod**. This lab walks the three rungs of that ladder, from throwaway scratch space to real dynamically-provisioned disks.

> Standalone lab. It uses its own namespace `storage-demo`. Runs on the kind cluster from LAB000.

```
kubectl create ns storage-demo
```

## A. emptyDir - scratch space that dies with the pod
The simplest volume. `emptyDir` is a fresh directory created when the pod starts and **deleted when the pod is removed**. It lives on the node, and its main use is sharing files between containers in the same pod or as temporary scratch.
```
kubectl apply -n storage-demo -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: scratch
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh","-c","echo hello > /data/file; sleep 3600"]
    volumeMounts:
    - name: tmp
      mountPath: /data
  volumes:
  - name: tmp
    emptyDir: {}
EOF
```
Wait for it to be ready, read it back, then delete the pod and note the data is gone forever:
```
kubectl wait --for=condition=Ready pod/scratch -n storage-demo --timeout=60s
kubectl exec -n storage-demo scratch -- cat /data/file
```
```
kubectl delete pod scratch -n storage-demo
```
### Sharing between containers (init container + main)
The real use of `emptyDir` is **sharing a directory between containers in the same pod**. A classic pattern: an **init container** prepares content into the shared volume, then the **main container** serves it. Same volume, two containers, a clean handoff.
```
kubectl apply -n storage-demo -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: shared-web
spec:
  initContainers:
  - name: fetch-content            # runs first, writes into the shared volume
    image: busybox
    command: ["sh","-c","echo '<h1>Served from a shared emptyDir</h1>' > /work/index.html"]
    volumeMounts:
    - name: web
      mountPath: /work
  containers:
  - name: web                      # starts after init finishes, serves the file
    image: nginx:alpine
    volumeMounts:
    - name: web
      mountPath: /usr/share/nginx/html
      readOnly: true
  volumes:
  - name: web
    emptyDir: {}
EOF
```
The init container writes `index.html` into the `emptyDir`; nginx mounts the **same** volume and serves it. Prove it end to end:
```
kubectl wait --for=condition=Ready pod/shared-web -n storage-demo --timeout=60s
kubectl port-forward -n storage-demo pod/shared-web 8080:80 &
curl -s localhost:8080
```
You get back the HTML the init container produced, served by a different container, because they share the volume. In real life the init container might `git clone` or `curl` the content instead of echoing it. Stop the port-forward with `kill %1` when done.

`emptyDir` is not persistence, it is a shared RAM/disk tmp between containers. For anything you care about, keep reading.

## B. hostPath - a directory from the node
`hostPath` mounts a path from the **node's own filesystem** into the pod. The data survives the pod, but only on that one node.
```
kubectl apply -n storage-demo -f hostpath-pod.yaml
kubectl wait --for=condition=Ready pod/hostpath-demo -n storage-demo --timeout=60s
kubectl exec -n storage-demo hostpath-demo -- sh -c 'echo "written by pod" > /host/hello.txt; cat /host/hello.txt'
```

> **On kind (incl. Docker Desktop on Mac):** the "node" is a Docker container, so this path lives *inside* that node container, on a Mac that means inside Docker Desktop's Linux VM, **not** on your macOS disk. The data survives pod delete/recreate on the same node and a node-container restart, but it is wiped by `kind delete cluster` and is not visible from your Mac's filesystem. So it is node-container-scoped, not true host persistence.

Two things a network/security engineer must know about `hostPath`:

- **No scheduling awareness.** A plain `hostPath` puts **no constraint** on the scheduler. An in-place container restart stays on the node, but a *recreated* pod (Deployment replacing it, or a delete+create) is scheduled fresh and can land on a different node, where that directory is empty or missing. The data does not follow.
- **Security risk.** `hostPath` is direct access to the node filesystem, a classic container-escape / privilege-escalation vector, and is commonly blocked by Pod Security Standards. Use it only for node-level agents that genuinely need it.

If you need node-local data that the scheduler *keeps pinned* to the right node, use a `local` PersistentVolume instead (it carries node affinity), not `hostPath`.

## C. PVC / PV / StorageClass - dynamic provisioning
This is the real model. Three objects work together:

- **PersistentVolumeClaim (PVC)** - the app's *request*: "I need 1Gi, ReadWriteOnce." Pods reference the PVC, never the disk directly.
- **StorageClass (SC)** - the *how*: which provisioner creates the storage and with what parameters. It is the "when someone asks for storage, use this backend" profile.
- **PersistentVolume (PV)** - the actual piece of storage. With dynamic provisioning you never write this by hand, the provisioner creates it for you.

kind ships a default StorageClass. Look at it:
```
kubectl get storageclass
```
```
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  1h
```
`(default)` means a PVC that omits `storageClassName` uses this class. `WaitForFirstConsumer` means the PV is **not** created until a pod actually mounts the PVC, so the disk lands on the node the pod is scheduled to.

Create a claim:
```
kubectl apply -n storage-demo -f pvc.yaml
kubectl get pvc -n storage-demo
```
It sits `Pending` on purpose, that is `WaitForFirstConsumer` waiting for a pod. Now create a pod that mounts it, and wait for the pod to be Ready:
```
kubectl apply -n storage-demo -f pod-with-pvc.yaml
kubectl wait --for=condition=Ready pod/pvc-demo -n storage-demo --timeout=90s
kubectl get pvc -n storage-demo
kubectl get pv | grep storage-demo
```
Binding is **not instant**. With `WaitForFirstConsumer` the provisioner only creates the PV once the pod is scheduled, and the `standard` / `local-path` provisioner then takes a few seconds to run its helper. That is exactly why the `wait` matters, a `get` issued the moment after `apply` still shows `Pending`. Once the pod is Ready the PVC is `Bound` and a PV has appeared automatically. (PVs are **cluster-scoped**, so on a shared cluster `kubectl get pv` also lists other namespaces' volumes, hence the `grep`.) Write some data:
```
kubectl exec -n storage-demo pvc-demo -- sh -c 'echo "persistent data" > /data/state.txt; cat /data/state.txt'
```
Now prove it persists across the pod's life. Delete and recreate **the pod** (not the PVC):
```
kubectl delete pod pvc-demo -n storage-demo
```
```
kubectl apply -n storage-demo -f pod-with-pvc.yaml
kubectl wait --for=condition=Ready pod/pvc-demo -n storage-demo --timeout=90s
kubectl exec -n storage-demo pvc-demo -- cat /data/state.txt
```
The file is still there, because it lives in the PV, not the pod.

### What actually happened (the flow)
1. The pod referenced a **PVC**; the PVC named (implicitly) the default **StorageClass**.
2. Because of `WaitForFirstConsumer`, the provisioner waited until the pod was scheduled, then created a **PV** on that node and bound PVC to PV.
3. The kubelet mounted the PV into the container at `/data`.
4. Deleting the pod leaves the PVC and PV untouched, so a new pod re-mounts the same data.

## Explore it yourself
* `kubectl describe pvc pvc-demo -n storage-demo` - which provisioner and node did it bind to?
* Delete the **PVC** (not just the pod). With `reclaimPolicy: Delete`, what happens to the PV and the data?
* Set `storageClassName: ""` in the PVC and re-apply. Why does it now stay `Pending` forever? (Hint: empty string means "no class, static only".)
* On a cloud cluster `kubectl get sc` shows `ebs.csi.aws.com` / `disk.csi.azure.com` / `pd.csi.storage.gke.io` instead of `local-path`. Same PVC, different backend, that is the point of the abstraction.

## Cleanup
```
kubectl delete ns storage-demo
```

> Takeaway: `emptyDir` is scratch that dies with the pod; `hostPath` is node-local data with **no** scheduler pinning and a security cost; **PVC + StorageClass + PV** is the real model, where the pod requests storage by claim and a provisioner dynamically creates the backing disk. `WaitForFirstConsumer` delays that creation until scheduling so the disk lands in the right place.
