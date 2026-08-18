# LAB010 - Learning about pods

A **pod** is the smallest deployable unit in Kubernetes, one or more containers that share a single network namespace. In network terms: every pod gets its **own IP address** (from the pod CIDR you configured in LAB000), and containers inside a pod reach each other over `localhost`. Pod-to-pod traffic is routed flat, with **no NAT** between pods.

A **namespace** is a logical boundary for grouping and isolating resources, think of it like a tenant or VRF for your objects (it scopes names and, later, network policy).

## Namespaces
First, list the namespaces that already exist on the cluster:
```
kubectl get ns
```
You will see the built-in ones such as `default`, `kube-system`, and `kube-public`.

Create a namespace for this lab:
```
kubectl create ns prod-nginx
```

## Deploy a pod
Deploy a single pod into the new namespace, either from a file:
```
kubectl apply -f pod.yaml
```
or inline with a heredoc:
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  namespace: prod-nginx
  labels:
    name: nginx
    environment: prod
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
EOF
```

## Listing pods
`kubectl get po` lists pods, and the flags change which namespace and how much detail you see.

Pods in the `default` namespace:
```
kubectl get po
```
Nothing shows here, because you created the pod in `prod-nginx`, not `default`.

Pods in a specific namespace:
```
kubectl get po -n prod-nginx
```
This is where your `nginx-pod` lives.

Pods in every namespace:
```
kubectl get po -A
```
This also lists the control plane pods in `kube-system`.

## Pod details with -o wide
Add `-o wide` to see the pod's IP address and the node it is scheduled on:
```
kubectl get po -n prod-nginx -o wide
NAME        READY   STATUS    RESTARTS   AGE    IP              NODE          NOMINATED NODE   READINESS GATES
nginx-pod   1/1     Running   0          136m   10.10.162.130   kind-worker   <none>           <none>
```
The two interesting columns are `IP` (the pod's own address, from the pod CIDR) and `NODE` (which node it landed on).

## Pod labels with --show-labels
Add `--show-labels` to see the labels attached to the pod:
```
kubectl get po -n prod-nginx -o wide --show-labels
NAME        READY   STATUS    RESTARTS   AGE    IP              NODE          NOMINATED NODE   READINESS GATES   LABELS
nginx-pod   1/1     Running   0          138m   10.10.162.130   kind-worker   <none>           <none>            environment=prod,name=nginx
```
Labels look cosmetic now, but they are how Services and network policies will target this pod later.

## Throwaway and debug pods
A pod runs its container's command and, when that command finishes, the pod stops. So a bare `ubuntu` pod exits immediately, there is nothing keeping it running. There are two ways to get a shell to work in.

Run one interactively and attach right away. The `-it` attaches your terminal and `--rm` deletes the pod when you exit:
```
kubectl run tmp -it --rm -n prod-nginx --image ubuntu -- bash
```
This is a true throwaway, it is gone the moment you leave.

Or start a pod that stays up by giving it a long-running command like `sleep`, then exec into it as often as you like:
```
kubectl run debug -n prod-nginx --image ubuntu -- sleep infinity
kubectl exec -it -n prod-nginx debug -- bash
```
This one persists until you delete it (`kubectl delete po -n prod-nginx debug`), so you can exec in and out while it keeps running.

### Exercise
Work through these yourself, the interesting part is figuring out the "why".

* Create a new namespace `lab01-exercise` (namespace names can't contain underscores, so no `lab01_exercise`)
* Create an `nginx` pod in the new namespace
* Find the IP address of the new pod
* Start an interactive throwaway pod in the same namespace:
  `kubectl run tmp -it -n lab01-exercise --rm --image ubuntu -- bash`
* From inside it, try to `curl` the nginx pod's IP
* If it fails, work out why and fix it (what does a bare `ubuntu` image not ship with?)
* Exit the pod (`exit`)
* Start the ubuntu pod again. What do you see, and what does that tell you about a pod's filesystem?
* Now run the throwaway pod in a **different** namespace (for example `default`) and `curl` the same nginx pod IP over in `lab01-exercise`. Does it still work? What does that tell you about whether a namespace is a network boundary by default?

> Takeaway: pod IPs are ephemeral and a pod's filesystem resets on restart, so **Services** (next labs) give you a stable virtual IP in front of changing pods. And a namespace isolates names, not traffic, cross-namespace pod-to-pod reachability is open by default until you add a **NetworkPolicy** (LAB075).
