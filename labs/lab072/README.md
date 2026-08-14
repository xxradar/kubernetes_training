# LAB025 - Resource requests and limits (CPU, memory, QoS)

> **Standalone lab.** It uses its own namespace `resource-demo`, so the `ResourceQuota` at the end does not affect the `prod-nginx` chain. Follows naturally from LAB020 (Deployments).

Every container can declare two numbers per resource:

- **requests** - what the pod is guaranteed. The **scheduler** uses requests to place the pod (it reserves that much capacity on a node) and to bin-pack. Requests do not cap usage.
- **limits** - the hard ceiling. What happens when a container exceeds it depends on the resource:
  - **CPU is compressible**: over the limit the container is **throttled** (slowed down), not killed.
  - **memory is incompressible**: over the limit the container is **OOMKilled** and restarted.

CPU is measured in cores / millicores (`500m` = half a core), memory in bytes (`Mi`, `Gi`). For a security engineer this is also a **containment control**: limits stop a runaway or compromised pod from starving its neighbours (a local DoS).

```
kubectl create ns resource-demo
```

## A. CPU limit means throttling, not killing
This pod asks `stress` to burn 2 CPUs but is limited to `500m` (half a core):
```
kubectl apply -n resource-demo -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress","--cpu","2"]
    resources:
      requests: {cpu: "250m", memory: "64Mi"}
      limits:   {cpu: "500m", memory: "64Mi"}
EOF
```
The pod stays `Running` (CPU is just capped). Prove it is being throttled by reading the container's cgroup CPU stats:
```
kubectl exec -n resource-demo cpu-demo -- cat /sys/fs/cgroup/cpu.stat
```
Look at `nr_throttled` and `throttled_usec`, they climb because the process is repeatedly held back to stay under `500m`.

## B. Memory limit means OOMKill
This pod tries to allocate 200M but is limited to `100Mi`:
```
kubectl apply -n resource-demo -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: mem-demo
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress","--vm","1","--vm-bytes","200M","--vm-hang","1"]
    resources:
      requests: {cpu: "100m", memory: "50Mi"}
      limits:   {cpu: "200m", memory: "100Mi"}
EOF
```
Watch it get killed and restarted (memory cannot be throttled). The restart count climbs and the pod settles into `CrashLoopBackOff`:
```
kubectl get po -n resource-demo mem-demo -w
```
Confirm it was the memory limit by looking at the terminated container:
```
kubectl describe pod mem-demo -n resource-demo
```
```
kubectl get po -n resource-demo mem-demo -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}{" exit="}{.status.containerStatuses[0].lastState.terminated.exitCode}{" restarts="}{.status.containerStatuses[0].restartCount}{"\n"}'
```
The reliable proof of an OOM kill is **`Exit Code: 137`** (128 + 9, SIGKILL from the kernel OOM killer) together with a **climbing restart count**.

> **Gotcha: `OOMKilled` vs `Error`.** The `Reason` field may read `OOMKilled` on one node and the generic `Error` on another, for the exact same pod. On cgroup v2 it depends on which process the kernel OOM killer picks. `stress --vm 1` runs a parent (PID 1) plus a worker child; if the **child** is the victim, the runtime cannot attribute the kill to the container's init process and reports `Error` (still exit 137). You get `OOMKilled` when PID 1 is killed or the whole cgroup is killed together (`memory.oom.group`). So do not key on the word `OOMKilled`, trust **exit code 137 + rising restarts**.

## C. Quality of Service (QoS) classes
Kubernetes derives a **QoS class** from how you set requests and limits. It decides the **eviction order** when a node runs out of memory:

- **Guaranteed** - every container sets requests **equal to** limits for both CPU and memory. Evicted last.
- **Burstable** - has some requests/limits, but not all equal (like `cpu-demo` and `mem-demo` above).
- **BestEffort** - no requests or limits at all. Evicted first.

Create one of each and compare:
```
kubectl apply -n resource-demo -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-demo
spec:
  containers:
  - name: pause
    image: registry.k8s.io/pause:3.9
    resources:
      requests: {cpu: "100m", memory: "64Mi"}
      limits:   {cpu: "100m", memory: "64Mi"}
EOF
kubectl run besteffort-demo -n resource-demo --image registry.k8s.io/pause:3.9
```
```
for p in cpu-demo mem-demo guaranteed-demo besteffort-demo; do
  echo -n "$p: "; kubectl get pod -n resource-demo $p -o jsonpath='{.status.qosClass}'; echo
done
```
Under memory pressure the kubelet evicts in this order: **BestEffort first, then Burstable, Guaranteed last**. So "no limits" is not free, it makes your pod the first to die.

## D. Enforcing defaults: LimitRange and ResourceQuota
Relying on developers to set requests/limits does not scale. Two namespace-scoped objects enforce it.

A **LimitRange** sets **default** requests/limits for pods that omit them (and can set min/max):
```
kubectl apply -n resource-demo -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
EOF
```
Now a pod created with no resources inherits those values. Check:
```
kubectl run inherit-demo -n resource-demo --image registry.k8s.io/pause:3.9
kubectl get pod inherit-demo -n resource-demo -o jsonpath='{.spec.containers[0].resources}'; echo
```

A **ResourceQuota** caps the **total** requests/limits (and object counts) for the whole namespace:
```
kubectl apply -n resource-demo -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
spec:
  hard:
    requests.cpu: "2"
    requests.memory: "2Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"
    pods: "10"
EOF
kubectl get resourcequota team-quota -n resource-demo
```
Once a `ResourceQuota` that limits `requests.*`/`limits.*` exists, every new pod in the namespace **must** declare those resources, or it is rejected at admission. That is the security-posture angle: you can force every workload in a tenant namespace to be bounded.

## Explore it yourself
* Raise `cpu-demo`'s CPU limit to `2` and re-check `cpu.stat`. Does the throttling stop?
* Give `mem-demo` a `256Mi` limit. Does it still get killed (exit 137), or does it now fit?
* With the `ResourceQuota` in place, try to create a pod with no resources. What does the API server say?
* Which QoS class would you give a critical security agent, and why?

## Cleanup
```
kubectl delete ns resource-demo
```

> Takeaway: **requests** drive scheduling (reserved capacity); **limits** cap usage. CPU over-limit is **throttled**, memory over-limit is **OOMKilled**. The mix of the two sets the **QoS class**, which decides who gets evicted first under pressure (BestEffort, then Burstable, then Guaranteed). Use **LimitRange** for defaults and **ResourceQuota** to force every workload in a namespace to stay bounded.
