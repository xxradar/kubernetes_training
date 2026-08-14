# LAB075 - Kubernetes native network security policies

A **NetworkPolicy** is Kubernetes' built-in, pod-level firewall. 

- **Namespaced and label-selected.** A policy lives in a namespace and picks pods by label (`podSelector`). It affects only the pods it selects.
- **Allow-only, additive.** There is no "deny" rule. You select pods and list what is *allowed*; everything else to/from those pods is dropped. Multiple policies **union** their allow-lists.
- **Empty selector = everything.** `podSelector: {}` selects every pod in the namespace, the trick for a namespace-wide default.
- **The switch is `policyTypes`.** A pod is unrestricted for a direction until *some* policy with that `policyType` (`Ingress`/`Egress`) selects it. After that, only explicit `from`/`to` rules get through.
- **Both ends matter.** Traffic from A to B must be allowed by **B's ingress** *and* **A's egress** (whenever either side has a policy for that direction). You will watch this play out below: opening the server's ingress is not enough while the client's egress is still denied.
- **Enforcement is the CNI's job.** The API object is standard, but a **policy-capable CNI must actually enforce it** — the object exists either way, so a policy can silently do nothing. See the note below on which CNIs enforce it.

> These labs require a cluster whose CNI **enforces** NetworkPolicy (**Cilium** or **Calico** are the easy choices). The final section uses a Cilium-only CRD.

### Which CNIs enforce NetworkPolicy?
**Enforce it:** Cilium, Calico (and Canal = Calico policy + flannel networking), Antrea, Kube-router. <br>
**Ignore it (no policy engine):** `kindnet`, plain `flannel`, and the AWS VPC CNI *without* its policy feature enabled.

On managed clouds NetworkPolicy is supported but usually **off until you turn it on**:

- **EKS** — the AWS VPC CNI enforces it **natively** since VPC-CNI **v1.14.0** (EKS ≥ 1.25), enabled with `enable-network-policy-controller: "true"` (eBPF); or install Calico.
- **GKE** — enable the **Network Policy** add-on (Calico-based), or use **Dataplane V2**, which is Cilium/eBPF and enforces it by default.
- **AKS** — choose a policy engine at cluster creation: **Cilium** (Azure CNI Powered by Cilium), **Calico**, or **Azure NPM** (NPM is being retired on Linux by 30 Sep 2028 in favour of Cilium).

## Setting up a lab environment
Three namespaces: `prod-nginx` (the workload we protect), plus `dev-nginx` and `myhackns` (clients we test from).
```
kubectl create ns prod-nginx
kubectl create ns dev-nginx
kubectl create ns myhackns
```
Deploy nginx and a ClusterIP Service in `prod-nginx`:
```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: prod-nginx
  labels:
    app: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
        env: prod
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-clusterip
  namespace: prod-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
EOF
```

## Baseline: connectivity before any policy
Look at the pods, their labels, and the service:
```
kubectl get po -n prod-nginx -o wide --show-labels
kubectl get svc -n prod-nginx -o wide --show-labels
```
In your **first terminal**, grab a backend pod IP (this `$POD` value is passed into the debug pod so you can also test a raw pod IP):
```
POD=$(kubectl get pods -n prod-nginx  -l app=nginx -o jsonpath='{range .items[0]}{@.status.podIP}{"\n"}{end}')
```
Open a **second terminal** and start a throwaway client pod. Keep it open, network policies apply to running pods live, so you can watch each change take effect without restarting:
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon --env="POD=$POD" debug
```
Inside the pod, run the three probes we reuse at every step:
```
nslookup my-nginx-clusterip     # DNS  -> egress to kube-dns
curl my-nginx-clusterip         # via the Service -> nginx ingress + your egress
curl $POD                       # straight to a pod IP
```
**Expected (no policy yet):** all three succeed.

> **Tip:** `kubectl run debug` automatically labels the pod `run=debug`, that label is what the egress policy in step 4 selects.

## Network policies

### 1. Default-deny
The foundational move: select every pod, mark both directions restricted, allow nothing.
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
   - Ingress
   - Egress
EOF
```
```
kubectl get netpol -n prod-nginx
```
Re-run the probes from a fresh debug pod:
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon --env="POD=$POD" debug
```
```
nslookup my-nginx-clusterip
curl my-nginx-clusterip
curl $POD
```
**Expected:** everything fails, `nslookup` times out and both curls hang. `prod-nginx` is now sealed in both directions.

### 2. Allow DNS egress
Nothing works without name resolution, so first let every pod reach CoreDNS in `kube-system`. If your cluster did not already label `kube-system`, add the label the policy matches on:
```
kubectl label ns kube-system kubernetes.io/metadata.name=kube-system
```
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector:
    matchLabels: {}
  policyTypes:
   - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
      podSelector:
        matchLabels:
          k8s-app: kube-dns
EOF
```
```
kubectl get netpol -n prod-nginx
```
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon --env="POD=$POD" debug
```
```
nslookup my-nginx-clusterip
curl my-nginx-clusterip
curl $POD
```
**Expected:** `nslookup` now resolves, but both curls still fail. We allowed DNS egress only, reaching nginx needs the server's ingress *and* the client's egress (the next two steps).

### 3. Allow HTTP ingress (server-side)
Let nginx accept traffic on port 80 from any pod in the namespace:
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-http
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels: {}
    ports:
    - protocol: TCP
      port: 80
EOF
```
```
kubectl get netpol -n prod-nginx
```
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon --env="POD=$POD" debug
```
```
nslookup my-nginx-clusterip
curl my-nginx-clusterip
curl $POD
```
**Expected:** still blocked. The **server** side is open now, but the debug pod's **egress** is still denied (only DNS was allowed). Opening one side is not enough, which is exactly the point of the next step.

### 4. Allow HTTP egress (client-side)
Now allow the `debug` pod (label `run=debug`) to initiate egress to pods in the namespace:
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-debug-egress
spec:
  podSelector:
    matchLabels:
      run: debug
  egress:
  - to:
    - podSelector:
        matchLabels: {}
EOF
```
```
kubectl get netpol -n prod-nginx
```
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon --env="POD=$POD" debug
```
```
nslookup my-nginx-clusterip
curl my-nginx-clusterip
curl $POD
```
**Expected:** both curls now succeed. Ingress (server) **and** egress (client) are both satisfied.

### 5. Ingress from a different namespace
Cross-namespace traffic needs a `namespaceSelector`. Two equivalent options: extend `allow-http` with a second `from`, or add a dedicated policy.
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-http
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels: {}
    ports:
    - protocol: TCP
      port: 80
  - from:
    - namespaceSelector:
        matchLabels:
          project: debug
      podSelector:
        matchLabels:
          mode: debug
    ports:
    - protocol: TCP
      port: 80
EOF
```
or
```
kubectl apply -n prod-nginx -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-http-other-namespace
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          project: debug
      podSelector:
        matchLabels:
          mode: debug
    ports:
    - protocol: TCP
      port: 80
EOF
```
```
kubectl get netpol -n prod-nginx
```
Label the client namespace so the `namespaceSelector` matches, then launch a client in `myhackns` carrying the `mode=debug` label the policy requires:
```
kubectl label ns myhackns project=debug
```
```
kubectl run -it --rm  -n myhackns --image xxradar/hackon -l mode=debug debug
```
```
nslookup my-nginx-clusterip.prod-nginx
curl my-nginx-clusterip.prod-nginx
```
**Expected:** works, note the cross-namespace FQDN `my-nginx-clusterip.prod-nginx`. (There is no policy in `myhackns`, so its egress is unrestricted; only the `prod-nginx` ingress rule gates this.)

### 6. Selectors are precise (both labels must match)
Right namespace, **wrong pod label** (`mode=nodebug`), blocked:
```
kubectl run -it --rm  -n myhackns --image xxradar/hackon -l mode=nodebug debug
curl my-nginx-clusterip.prod-nginx
```
Right pod label, **wrong namespace** (`dev-nginx` is not labelled `project=debug`), blocked:
```
kubectl run -it --rm  -n dev-nginx --image xxradar/hackon -l mode=debug debug
curl my-nginx-clusterip.prod-nginx
```
Fix it by labelling the namespace, then retry:
```
kubectl label ns dev-nginx project=debug
```
```
kubectl run -it --rm  -n dev-nginx --image xxradar/hackon -l mode=debug debug
```
```
curl my-nginx-clusterip.prod-nginx
```
**Expected:** blocked, blocked, then allowed. Both the **namespace** label and the **pod** label must match, precise, least-privilege selection.

## Advanced: Cilium cluster-wide policy (quarantine)
Standard NetworkPolicy is namespaced. Cilium adds a **cluster-wide** CRD, useful for a security response like quarantining a compromised pod anywhere in the cluster. This one denies all egress to the outside `world` for any pod labelled `quarantine=true`:
```
kubectl apply -f - <<EOF
apiVersion: "cilium.io/v2"
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: "quarantine"
spec:
  endpointSelector:
    matchLabels:
      quarantine: "true"
  egressDeny:
  - toEntities:
    - "world"
EOF
```
Start a client and confirm external access works first:
```
kubectl run -it --rm -n myhackns --image xxradar/hackon --env="POD=$POD" debug
```
```
nslookup www.radarhack.com
curl https://www.radarhack.com
```
In another terminal, quarantine the running pod:
```
kubectl label po/debug -n myhackns  quarantine=true
```
Return to the pod and try again:
```
curl https://www.radarhack.com
curl https://www.radarhack.com
```
**Expected:** the external call succeeds before the label and fails the instant `quarantine=true` is applied, live, on the already-running pod. (Cilium only; Calico's equivalent is `GlobalNetworkPolicy`.)

## Cleanup
```
kubectl delete ns prod-nginx
kubectl delete ns dev-nginx
kubectl delete ns myhackns
kubectl delete CiliumClusterwideNetworkPolicy quarantine
```

> Takeaway: NetworkPolicy is an **allow-only, label-selected, namespaced** pod firewall. `podSelector: {}` + `policyTypes` gives you a namespace **default-deny**, and from there you additively allow what is needed, remembering traffic must clear **both** the server's ingress and the client's egress. Enforcement needs a policy-capable CNI (Cilium/Calico); Cilium's cluster-wide CRD extends the same idea beyond a single namespace.
