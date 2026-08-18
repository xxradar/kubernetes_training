# LAB085 - Observability: events, logs, metrics, and audit

When something misbehaves, four signals answer four different questions:

- **Events** - what the control plane *did* (scheduling, image pulls, probe failures, evictions).
- **Logs** - what the *application* said.
- **Metrics** - how much *resource* is being used.
- **Audit log** - who called the *API*, and what they asked for.

This lab is a quick tour of each on the LAB000 cluster.

> Runs on the kind + Cilium cluster from LAB000. Some steps assume the `prod-nginx` workload from earlier labs; any Deployment works.

## A. Events
Events are the cluster's short-term memory (kept ~1 hour by default). List them newest-last:
```
kubectl get events -A --sort-by=.lastTimestamp | tail -20
```
Scope to one namespace and watch them live:
```
kubectl get events -n prod-nginx --watch
```
`kubectl describe` shows the events for a single object at the bottom, the first place to look when a pod is stuck:
```
kubectl describe pod -n prod-nginx <pod> | sed -n '/Events:/,$p'
```
**What to look for:** `Scheduled`, `Pulling`/`Pulled`, `Started`, and the bad ones, `FailedScheduling`, `BackOff`, `Unhealthy` (a failing probe), `OOMKilling`.

## B. Logs
Application stdout/stderr:
```
kubectl logs -n prod-nginx <pod>
```
Useful flags:
```
kubectl logs -n prod-nginx <pod> -f              # follow (tail -f)
kubectl logs -n prod-nginx <pod> --since=10m     # last 10 minutes
kubectl logs -n prod-nginx <pod> --previous      # the PREVIOUS container (after a crash/restart)
kubectl logs -n prod-nginx deploy/nginx-deployment --all-containers  # across the deployment's pods
```
`--previous` is the one that matters for `CrashLoopBackOff`: the current container may be too young to have logged anything, the crash reason is in the previous instance.

## C. Metrics (metrics-server and `kubectl top`)
`kubectl top` needs **metrics-server**, which is not installed by default on kind. Install it, then allow the kubelet's self-signed cert (kind-only):
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl -n kube-system patch deployment metrics-server --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
kubectl -n kube-system rollout status deployment metrics-server
```
Now read live CPU/memory:
```
kubectl top nodes
kubectl top pods -A
```
**Expected:** a table of CPU (millicores) and memory (Mi) per node/pod. This is the same data the Horizontal Pod Autoscaler and the scheduler's pressure signals rely on.

> **The Prometheus model (context).** metrics-server holds only a short, in-memory window for `top` and autoscaling. For history, alerting, and app metrics you run **Prometheus**, which *pulls* (scrapes) a `/metrics` endpoint off each target on an interval and stores time series. Same idea, different scale, not installed here.

## D. Network flow visibility with Hubble (Cilium)
Metrics and logs stop at the pod boundary; **Hubble** (part of Cilium, enabled in LAB000) shows the actual **network flows**, which identity talked to which, on what port, and whether policy allowed or dropped it. This is where LAB075's NetworkPolicy results become visible.
```
cilium hubble enable            # if not already on
hubble status
```
Observe flows (via the Hubble CLI, port-forwarded by `cilium hubble port-forward`):
```
hubble observe -n prod-nginx --last 20
hubble observe -n prod-nginx --verdict DROPPED
```
`--verdict DROPPED` is the fast way to confirm a NetworkPolicy is doing its job (or catch one blocking traffic you wanted). `cilium hubble ui` opens the service map in a browser.

## E. Audit logs (who called the API)
The **audit log** records every request to the API server: the user/ServiceAccount, verb, resource, and response. It is the record you reach for after an incident ("who deleted that Deployment?"). It is configured on the **control plane** with an audit policy and a log backend; the LAB000 kind setup already enables API-server audit logging.

On a kind cluster the API server runs as a static pod, so the log lives on the control-plane node:
```
docker exec kind-control-plane sh -c 'ls -la /var/log/kubernetes/ 2>/dev/null; tail -n 5 /var/log/kubernetes/kube-apiserver-audit.log 2>/dev/null'
```
Each line is a JSON event with `user.username`, `verb`, `objectRef` (resource/namespace/name), and `responseStatus`. On managed clusters you read the same data through the cloud's logging service (CloudWatch, Azure Monitor, Cloud Logging) rather than a file.

## Explore it yourself
* Break a probe (LAB032) and watch the `Unhealthy` events appear with `kubectl get events -w`.
* Run a CPU-hungry pod (LAB072's `cpu-demo`) and watch it in `kubectl top pods`.
* Apply a default-deny (LAB075) and confirm the drop in `hubble observe --verdict DROPPED`.
* In the audit log, filter for a specific verb: which identity issued the last `delete`?

## Cleanup
```
# metrics-server can stay; to remove it:
kubectl delete -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

> Takeaway: four signals, four questions. **Events** show what the control plane did, **logs** show what the app said (use `--previous` for crashes), **metrics** (metrics-server / `kubectl top`, Prometheus for history) show resource use, **Hubble** shows allowed vs dropped network flows, and the **audit log** shows who called the API. Knowing which one answers your question is most of troubleshooting.
