# LAB063 - Ingress (the classic L7 entry point)

An **Ingress** is Kubernetes' original way to expose HTTP(S) services to the outside world at **L7**: one shared entry point that routes by **Host header** and **URL path** to different backend Services, with optional TLS termination. The Ingress *resource* is only the rules; an **ingress controller** (nginx, Traefik, HAProxy, or here Cilium's built-in one) is the actual proxy that enforces them.

Think of it as a shared reverse proxy / L7 virtual server. A LoadBalancer Service (LAB050) gives **one** service **one** external L4 IP; an Ingress lets **many** HTTP services share a single entry point and splits them by Host/path, like name-based virtual hosting on a web proxy.

> Why "classic": Ingress is the older API. It works and is everywhere, but it is limited and is being superseded by the **Gateway API** (LAB065). We cover it so you recognise it, then show the modern model next.

> Standalone lab. Namespace `ingress-demo`. Runs on the kind cluster from LAB000.

## No controller to install
Our Cilium install in LAB000 already enabled the built-in ingress controller (`ingressController.enabled=true`, shared mode, NodePort service), so there is nothing extra to deploy. Confirm the ingress class exists:
```
kubectl get ingressclass
```
```
NAME     CONTROLLER                     PARAMETERS   AGE
cilium   cilium.io/ingress-controller   <none>       1h
```

## Deploy an app
A tiny echo server plus a ClusterIP Service (internal only, the Ingress will publish it):
```
kubectl create ns ingress-demo
kubectl apply -n ingress-demo -f ingress-webapp.yaml
kubectl wait --for=condition=Available deploy/echo -n ingress-demo --timeout=90s
kubectl get pods,svc -n ingress-demo
```

## Create the Ingress
The rule says: traffic for Host `echo.local`, any path, goes to the `echo` Service. Note `ingressClassName: cilium`, that is what binds this Ingress to Cilium's controller.
```
kubectl apply -n ingress-demo -f ingress.yaml
kubectl get ingress -n ingress-demo
```

## Reach it
In shared mode Cilium publishes **one** service for all Ingresses, `cilium-ingress` in `kube-system`. Because the install set the service type to NodePort, grab its node port for 80:
```
kubectl get svc -n kube-system cilium-ingress
```
```
export NODE=$(kubectl get no kind-worker -o jsonpath='{.status.addresses[0].address}')
export IPORT=$(kubectl get svc -n kube-system cilium-ingress -o jsonpath='{.spec.ports[?(@.port==80)].nodePort}')
```
Curl the node port with the matching `Host` header (the Host is how the Ingress picks the rule):
```
curl -H "Host: echo.local" http://$NODE:$IPORT/
```
> On a KIND cluster the node IPs live on the `kind` docker network. If your shell can't reach `$NODE` directly, run the client on that network like in LAB040: `docker run --rm --network kind curlimages/curl -H "Host: echo.local" http://$NODE:$IPORT/`.

Without the right `Host` header you get a 404 from the controller, proof that routing is host-based, not IP-based.

### Optional: a real external IP via MetalLB
The NodePort above comes from the LAB000 install (`ingressController.service.type=NodePort`), the safe default on kind where there is no cloud load balancer. If you set up **MetalLB** in LAB050, you can instead give the shared ingress a LoadBalancer IP from the MetalLB pool. This is also a clean example of using `helm upgrade` to tweak a **running** install: `--reuse-values` keeps every current setting and you override just one:
```
helm upgrade cilium cilium/cilium --version 1.19.6 -n kube-system --reuse-values \
  --set ingressController.service.type=LoadBalancer
kubectl -n kube-system rollout restart deploy/cilium-operator
```
Now `cilium-ingress` gets an `EXTERNAL-IP` from the MetalLB range, so you curl it directly, no node port needed:
```
export LBIP=$(kubectl get svc -n kube-system cilium-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -H "Host: echo.local" http://$LBIP/
```

## What is "old" about Ingress
Three limits push people to the Gateway API:

- **No role separation.** One `Ingress` kind mixes the entry-point (infra) and the routing (app). There is no clean split between the platform team that owns the shared proxy and the app teams that own their own routes.
- **Thin expressiveness.** The spec covers Host + path and little else. Header-based routing, traffic splitting / canary, gRPC, TLS passthrough, cross-namespace backends, all of it lives in **controller-specific annotations** that differ per controller and don't port between nginx, Traefik, Cilium, cloud LBs, etc.
- **Not typed / not portable.** Those annotations are opaque strings, not first-class API objects, so tooling and RBAC can't reason about them.

The **Gateway API** (LAB065) fixes exactly this: `GatewayClass` (the vendor/infra), `Gateway` (listeners / the VIP, owned by the platform team), and typed routes (`HTTPRoute`, `TLSRoute`, …, owned by app teams), portable, role-oriented, and expressive without annotations. Same job, cleaner model. That's the next lab.

## Explore it yourself
* Add a second Host (or path) in `ingress.yaml` pointing at another Service. One entry point, many apps, split by Host, that is the whole point of Ingress.
* Look again at `cilium-ingress` (shared mode): how would "dedicated" mode differ (one LB per Ingress)?
* What would **header-based** routing (e.g. `X-Canary: true`) require here? (Answer: a controller-specific annotation.) Keep that in mind for LAB065, where it is a first-class `HTTPRoute` match.

## Cleanup
```
kubectl delete ns ingress-demo
```

> Takeaway: an Ingress is a shared L7 entry point that routes by Host/path to Services, implemented by an ingress controller (here Cilium's built-in one, already enabled in LAB000). It is the classic, annotation-driven approach with no role separation; the Gateway API in LAB065 is its modern, typed, role-oriented successor.
