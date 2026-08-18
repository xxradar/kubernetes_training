# LAB100 - Troubleshooting

A practical toolbox for figuring out why a pod is unhappy or where traffic is going. Most problems are solved by a short chain: **describe** the object (read its events), **read the logs**, then **get inside** the pod or its network namespace to look around.

> Uses the `prod-nginx` workload from earlier labs; any pod works. Replace pod names with your own.

## Triage: describe first, it carries the events
`kubectl describe` shows an object's current state and its recent **events** at the bottom, which usually name the problem outright.
```
kubectl get po -n prod-nginx
kubectl describe pod -n prod-nginx <pod> | sed -n '/Events:/,$p'
```
Common patterns and where to look next:

- **Pending** - `FailedScheduling` in events (no node fits: resources, taints, affinity). See LAB037/LAB072.
- **ImagePullBackOff** - wrong image name or missing pull secret; the event says which.
- **CrashLoopBackOff** - the app starts and dies; read the **previous** container's logs (below).
- **0/1 Ready** but Running - a failing readiness probe; the event is `Unhealthy`. See LAB032.

## Read the logs
```
kubectl logs -n prod-nginx <pod>
kubectl logs -n prod-nginx <pod> -f            # follow
kubectl logs -n prod-nginx <pod> --previous    # the container BEFORE the last restart
kubectl logs -n prod-nginx deploy/nginx-deployment --all-containers --tail=20
```
For a `CrashLoopBackOff`, `--previous` is the important one: the running container may be too young to have logged the failure.

## Execute commands inside a pod
Drop into a shell (or run a one-off command) in a running container:
```
kubectl exec -it -n prod-nginx <pod> -- bash
# one-off, no shell:
kubectl exec -n prod-nginx <pod> -- cat /etc/nginx/conf.d/default.conf
```
From inside you can check config, DNS (`nslookup`, `cat /etc/resolv.conf`), and reachability (`curl`).

## Debug a pod that has no shell (ephemeral container)
Distroless / scratch images have no `bash`, `curl`, or `nslookup`. `kubectl debug` attaches a **temporary container** into the running pod, sharing its network and process namespaces, so you bring your own tools without rebuilding the image:
```
kubectl debug -it -n prod-nginx <pod> --image=xxradar/hackon --target=<container-name>
```
Inside, `localhost` is the target pod, so you can `curl localhost:80`, inspect `/proc`, and run network tools against the pod as it runs.

## Look at the node's network namespace (hostNetwork pod)
To see what the **node** sees (all interfaces, the CNI plumbing, real traffic), run a privileged debug pod in the host network namespace:
```
kubectl run -it --rm debug --restart=Never --image=xxradar/hackon \
  --overrides='{"kind":"Pod","apiVersion":"v1","spec":{"hostNetwork":true}}'
```
Now `ip a` shows the node's interfaces (the `cali*`/`cilium*` veth pairs, the CNI overlay device, `eth0`):
```
ip a
```
```
...
4: vxlan.calico: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 ...
    inet 10.10.110.128/32 scope global vxlan.calico
83: eth0@if84: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 172.18.0.5/16 brd 172.18.255.255 scope global eth0
...
```
And capture live traffic on the node with `tcpdump`:
```
tcpdump -i any -n
```
```
21:42:26.770551 IP 172.18.0.3.37542 > 172.18.0.5.10250: Flags [.], ack 541207, ...
21:42:26.770985 IP 172.18.0.5.10250 > 172.18.0.3.37542: Flags [P.], seq 542681:542711, ...
```
Filter to a pod IP or port to follow a specific flow, e.g. `tcpdump -i any -n host 172.18.0.5 and port 80`.

## Cluster-wide quick checks
```
kubectl get pods -A -o wide | grep -vE 'Running|Completed'   # anything not healthy
kubectl get events -A --sort-by=.lastTimestamp | tail -20    # recent cluster events
kubectl get nodes -o wide                                    # node readiness / IPs
kubectl top pods -A                                          # resource hot spots (needs metrics-server, LAB085)
```

## Explore it yourself
* Break a pod on purpose (bad image, failing probe from LAB032) and walk the chain: `describe` -> events -> `logs --previous`.
* Use `kubectl debug` against a running pod and `curl localhost` its own port. Why does that work without exposing a Service?
* From a `hostNetwork` debug pod, `tcpdump` port 80 while you curl a NodePort (LAB040). Can you see the SNAT from `externalTrafficPolicy: Cluster`?

> Takeaway: troubleshooting is a chain, not a guess. **describe** (events) tells you what the cluster thinks is wrong, **logs** (with `--previous` for crashes) tell you what the app said, and **exec** / **kubectl debug** / a **hostNetwork** pod let you look from inside the container, inside a shell-less pod, and from the node's own network namespace.
