# LAB074 - Hardening pods with securityContext

By default a container runs with far more privilege than it needs: as **root**, with a **writable** filesystem, a set of Linux **capabilities**, and the ability to **escalate** privileges. If that container is compromised, all of that becomes the attacker's. A pod's **`securityContext`** is where you strip those privileges away, shrinking the blast radius of a break-in. This is the core container-hardening checklist.

There are two levels, and the distinction matters:

- **Pod-level `securityContext`** (`spec.securityContext`) - identity and defaults for all containers: `runAsNonRoot`, `runAsUser`/`runAsGroup`, `fsGroup`, `seccompProfile`.
- **Container-level `securityContext`** (`spec.containers[].securityContext`) - per-container hardening: `capabilities`, `allowPrivilegeEscalation`, `readOnlyRootFilesystem`, `privileged`. Container-level wins over pod-level where they overlap.

> Standalone lab. Namespace `secctx-demo`. Runs on any cluster (kind is fine).

```
kubectl create ns secctx-demo
```

## Setup: an insecure and a hardened pod
Deploy the same image two ways, a bare pod with **no** `securityContext`, and one with the full checklist applied. We compare them field by field.
```
kubectl apply -n secctx-demo -f insecure-pod.yaml
kubectl apply -n secctx-demo -f hardened-pod.yaml
kubectl wait --for=condition=Ready pod/insecure pod/hardened -n secctx-demo --timeout=60s
```
Most of the checks below read the kernel's view of the container's PID 1 from `/proc/1/status`, an easy, image-agnostic way to see what actually took effect.

## A. Run as non-root (`runAsNonRoot` / `runAsUser`)
Running as UID 0 means a container escape lands the attacker as root on the node's namespaces. Check who each container runs as:
```
kubectl exec -n secctx-demo insecure -- id
kubectl exec -n secctx-demo hardened -- id
```
**Expected:** `insecure` is `uid=0(root)`; `hardened` is `uid=1000` (set by `runAsUser`). `runAsNonRoot: true` is the belt-and-braces: it tells the kubelet to **refuse to start** the container if it would run as root.

See that enforcement directly, a pod that demands non-root but whose image defaults to root and sets no UID:
```
kubectl apply -n secctx-demo -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: needs-nonroot
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","sleep 3600"]
EOF
kubectl get pod needs-nonroot -n secctx-demo
kubectl describe pod needs-nonroot -n secctx-demo | grep -i nonroot
```
**Expected:** it never starts, `CreateContainerConfigError`, with the message *"container has runAsNonRoot and image will run as root"*. The policy caught a root image at admission-to-runtime.

## B. Read-only root filesystem (`readOnlyRootFilesystem`)
A writable root filesystem lets an attacker drop tools, tamper with binaries, or persist. Make it read-only and give the app an explicit writable scratch dir (`emptyDir` at `/tmp`) instead:
```
kubectl exec -n secctx-demo insecure -- sh -c 'touch /root-write && echo "insecure: wrote to /"'
kubectl exec -n secctx-demo hardened -- sh -c 'touch /root-write || echo "hardened: / is read-only"'
kubectl exec -n secctx-demo hardened -- sh -c 'touch /tmp/ok && echo "hardened: /tmp is writable"'
```
**Expected:** the insecure pod writes anywhere; the hardened pod's write to `/` fails with *Read-only file system*, but `/tmp` (the mounted `emptyDir`) still works.

## C. Drop capabilities and block privilege escalation
Linux **capabilities** are slices of root's power (bind low ports, change file ownership, load modules…). Most apps need **none**. Drop them all, and set `allowPrivilegeEscalation: false` so a process can never gain more privilege than it started with (this sets the kernel's `no_new_privs`, which also neutralises setuid binaries).
```
kubectl exec -n secctx-demo insecure -- grep -E 'CapEff|NoNewPrivs' /proc/1/status
kubectl exec -n secctx-demo hardened -- grep -E 'CapEff|NoNewPrivs' /proc/1/status
```
**Expected:** the insecure pod has a non-zero `CapEff` (a default capability set) and `NoNewPrivs: 0`; the hardened pod shows `CapEff: 0000000000000000` (no capabilities) and `NoNewPrivs: 1`.

> **Never `privileged: true`** unless you truly mean it. A privileged container disables almost all of this isolation at once (full device access, all capabilities), it is effectively root on the node. It is the single field an attacker most wants to see set.

## D. seccomp (`seccompProfile: RuntimeDefault`)
**seccomp** filters which **syscalls** the container may make. `RuntimeDefault` applies the container runtime's curated blocklist of dangerous/legacy syscalls, a big reduction in kernel attack surface for near-zero effort. Check the seccomp mode of PID 1:
```
kubectl exec -n secctx-demo insecure -- grep Seccomp /proc/1/status
kubectl exec -n secctx-demo hardened -- grep Seccomp /proc/1/status
```
**Expected:** the hardened pod shows `Seccomp: 2` (seccomp **filter** active); the insecure pod shows `Seccomp: 0` (**Unconfined**, no filter) unless your cluster already defaults to `RuntimeDefault`.

## Putting it together
`hardened-pod.yaml` is the reusable template, the "restricted" baseline every workload should aim for:
```
kubectl get pod hardened -n secctx-demo -o jsonpath='{.spec.securityContext}{"\n"}{.spec.containers[0].securityContext}{"\n"}'
```
This is exactly the shape the **Pod Security Standards** `restricted` profile enforces (Session 5), non-root, no privilege escalation, all capabilities dropped, `RuntimeDefault` seccomp. Setting it per-pod is step one; admission control makes it mandatory cluster-wide.

## Explore it yourself
* Give the hardened container back one capability it might legitimately need, e.g. `add: ["NET_BIND_SERVICE"]` to bind port 80 as non-root. Re-check `CapEff`.
* Remove the `emptyDir` at `/tmp` from the hardened pod. Does an app that writes temp files still start? Where should writable paths come from?
* Try `privileged: true` on a container and inspect `CapEff`, what changed, and why is that dangerous on a shared node?
* Which of these four controls would have limited a real container-escape CVE you have read about?

## Cleanup
```
kubectl delete ns secctx-demo
```

> Takeaway: `securityContext` is the per-pod hardening checklist, run as **non-root**, **read-only** root filesystem, **drop all capabilities**, **no privilege escalation**, and **`RuntimeDefault` seccomp**, with pod-level fields for identity and container-level fields for per-container limits. Each one shrinks what a compromised container can do; together they are the `restricted` Pod Security Standard you will enforce cluster-wide in Session 5.
