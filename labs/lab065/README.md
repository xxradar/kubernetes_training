# LAB065 - Gateway API (Envoy Gateway)

The **Gateway API** is the modern successor to Ingress (LAB063). Same job, expose HTTP(S) services at L7, but a cleaner, **role-oriented** model built from typed objects instead of controller-specific annotations:

- **GatewayClass** - which controller/implementation handles gateways (the "vendor"). Cluster-scoped, set by the platform.
- **Gateway** - the listeners and the external address (the VIP / entry point). Owned by the **platform/infra team**.
- **HTTPRoute** (and `TLSRoute`, `GRPCRoute`, …) - the actual routing rules. Owned by the **app teams**, in their own namespaces, attached to a Gateway.

That separation is the whole point: infra owns the entry point, app teams own their routes, and features like traffic-splitting or header matching are **first-class fields**, not opaque annotations.

We use **Envoy Gateway** as the implementation. The Gateway is exposed through a LoadBalancer Service, so we reuse the **MetalLB** pool from LAB050 to give it an external IP.

> Standalone lab. Runs on the kind + Cilium cluster from LAB000. **Requires the MetalLB LoadBalancer from LAB050** so the Gateway gets an external IP. Uses namespaces `envoy-gateway-system` (the Gateway) and `demo-app` (the route + app).

## Prerequisite: a LoadBalancer (MetalLB from LAB050)
Envoy Gateway exposes the Gateway through a Service of type `LoadBalancer`, so the cluster needs something to hand it an external IP. On KIND that is the MetalLB pool from LAB050. Confirm it is present:
```
kubectl get ipaddresspool -n metallb-system
```
If you skipped LAB050, install MetalLB and its pool first (see LAB050). On a managed cloud (EKS/AKS/GKE) the cloud load balancer fills this role automatically, nothing to install.

## 1. Install the Gateway API CRDs
The API types are not built in; install the **standard channel** (GatewayClass, Gateway, HTTPRoute, …):
```
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
kubectl get crds | grep gateway.networking.k8s.io
```

## 2. Install Envoy Gateway
Envoy Gateway is the controller that watches Gateway/HTTPRoute objects and programs actual Envoy proxies. Install it with Helm (skip CRDs, we just installed the Gateway API ones):
```
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.7.3 -n envoy-gateway-system --create-namespace --skip-crds
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available
```

## 3. Create the GatewayClass
The GatewayClass points at the Envoy Gateway controller (the "pick your vendor" step):
```
kubectl apply -f gatewayclass.yaml
kubectl get gatewayclass
```
```
NAME   CONTROLLER                                      ACCEPTED   AGE
eg     gateway.envoyproxy.io/gatewayclass-controller   True       5s
```
`ACCEPTED: True` means the controller recognised it.

## 4. Create the Gateway
This defines an HTTP listener on port 80 and, via `allowedRoutes`, lets routes from any namespace attach. Envoy Gateway creates a LoadBalancer Service for it, which **MetalLB** assigns an IP from the LAB050 pool:
```
kubectl apply -f gateway.yaml
kubectl wait --timeout=5m -n envoy-gateway-system gateway/eg-gateway --for=condition=Programmed
kubectl get gateway -n envoy-gateway-system
```
The `ADDRESS` column shows the external IP (e.g. `172.18.255.200`). If it stays empty, MetalLB isn't handing out an IP, recheck the prerequisite above.

## 5. Deploy the app and attach a route
Create the app namespace, then a simple echo app plus its Service, and an **HTTPRoute** that attaches to the Gateway and sends `webapp.local.dev` to the Service:
```
kubectl create namespace demo-app
kubectl apply -f webapp.yaml
kubectl wait --for=condition=Available deploy/webapp -n demo-app --timeout=90s
kubectl apply -f httproute.yaml
kubectl get httproute -n demo-app
```

## 6. Test it
Grab the Gateway's address and curl it with the matching Host header:
```
export GW=$(kubectl get gateway eg-gateway -n envoy-gateway-system -o jsonpath='{.status.addresses[0].value}')
curl -H "Host: webapp.local.dev" http://$GW/
```
You should get `Hello from Gateway API! Pod: webapp-...`. Repeat and watch the pod name change as it load-balances.
> On KIND the LB IP lives on the `kind` docker network. If your shell can't reach it directly, run the client on that network: `docker run --rm --network kind curlimages/curl -H "Host: webapp.local.dev" http://$GW/`.

## 7. Add TLS termination (cert + Secret + HTTPS listener)
So far the Gateway only speaks HTTP. A real entry point terminates TLS. This step ties three things together: a **certificate**, a Kubernetes **TLS Secret** (LAB062) that stores it, and an **HTTPS listener** on the Gateway that references that Secret.

Generate a self-signed cert and key (in production this comes from a real CA or cert-manager, not `openssl`):
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key -out /tmp/tls.crt \
  -subj "/CN=*.local.dev" \
  -addext "subjectAltName=DNS:*.local.dev,DNS:localhost"
```
Store it as a **TLS Secret** in the Gateway's namespace. This is a typed `kubernetes.io/tls` Secret holding `tls.crt` + `tls.key`, the same Secret machinery from LAB062, just a specific shape the Gateway knows how to read:
```
kubectl create secret tls eg-tls-cert \
  --cert=/tmp/tls.crt --key=/tmp/tls.key \
  -n envoy-gateway-system
```
Now update the Gateway to add an HTTPS listener that terminates TLS with that Secret (`gateway-tls.yaml` keeps the `:80` listener and adds `:443` with `tls.mode: Terminate` and `certificateRefs: eg-tls-cert`):
```
kubectl apply -f gateway-tls.yaml
kubectl wait --timeout=5m -n envoy-gateway-system gateway/eg-gateway --for=condition=Programmed
```
Test over HTTPS. The existing HTTPRoute attaches to the new listener automatically; `-k` because the cert is self-signed:
```
curl -k -H "Host: webapp.local.dev" https://$GW/
```
Envoy decrypts at the Gateway (**TLS termination**) and forwards plain HTTP to the pod, exactly the SSL-offload pattern of a hardware load balancer. `mode: Terminate` means "decrypt here"; `mode: Passthrough` (via a `TLSRoute`) would instead forward the still-encrypted stream to the backend for end-to-end TLS.

## How this differs from Ingress (LAB063)
- **Roles are separated.** The `Gateway` (infra) and the `HTTPRoute` (app team, in `demo-app`) are distinct objects with distinct RBAC. In Ingress it was one blob.
- **Routing is typed, not annotated.** Header matches, traffic splitting, method matches, cross-namespace refs are **fields** in the spec. In Ingress each of those needed a controller-specific annotation.
- **Portable.** Swap Envoy Gateway for another Gateway API implementation and the `Gateway`/`HTTPRoute` objects stay the same, only the `GatewayClass` changes.

## Explore it yourself
* **Traffic split / canary:** add a second Deployment (`version: v2`) and a second `backendRefs` entry in the HTTPRoute with `weight: 10` vs `weight: 90`. No annotations, just weights.
* **Header match:** add a rule that matches header `X-Canary: true` and routes it to v2. Compare that to what LAB063's Ingress would have needed.
* **Second route:** add another HTTPRoute (different hostname) attached to the **same** Gateway, that is the shared-entry-point / role-split model in action.
* **TLS passthrough:** instead of terminating at the Gateway, use a `TLSRoute` with `mode: Passthrough` so the backend does its own TLS (end-to-end encryption). What can the Gateway still see, and what can't it?

## Cleanup
```
kubectl delete namespace demo-app
kubectl delete -f gateway-tls.yaml -f gatewayclass.yaml
kubectl delete secret eg-tls-cert -n envoy-gateway-system
helm uninstall eg -n envoy-gateway-system
```
(You can leave the Gateway API CRDs and MetalLB in place for later labs.)

> Takeaway: the Gateway API replaces Ingress with a typed, role-oriented model, `GatewayClass` (vendor), `Gateway` (infra's listeners + VIP), `HTTPRoute` (app team's rules). Here Envoy Gateway is the implementation and MetalLB (from LAB050) provides the external IP. Advanced routing (canary, header match, cross-namespace) is built into the API instead of hiding in controller-specific annotations, and TLS termination is just an HTTPS listener referencing a TLS Secret.
