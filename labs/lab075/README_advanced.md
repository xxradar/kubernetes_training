## Kubernetes native network security policies
These labs require a K8S cluster with Cilium or Calico CNI installed. <br>
For a quickstart, check out:<br>
https://learn.microsoft.com/en-gb/azure/aks/azure-cni-powered-by-cilium<br>
For overlay mode, also complete and read https://learn.microsoft.com/en-us/azure/aks/azure-cni-overlay.

### Setting up a lab environment
```
kubectl create ns prod-nginx
kubectl create ns dev-nginx
kubectl create ns myhackns
```
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
### Check connectivity
```
kubectl get po -n prod-nginx -o wide  --show-labels
...
kubectl get svc -n prod-nginx -o wide --show-labels
...
```
For the sake of simplicity, open a **second terminal**
```
POD=$(kubectl get pods -n prod-nginx  -l app=nginx -o jsonpath='{range .items[0]}{@.status.podIP}{"\n"}{end}')
```
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon --env="POD=$POD" debug
```
Inside the pod (you can keep it open, because network policies are applied on running pods)
```
nslookup my-nginx-clusterip
curl my-nginx-clusterip
curl $POD
```
## Network policies
### Default-deny
Apply a default-deny all policy
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
Check connectivity
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon --env="POD=$POD" debug
```
```
nslookup my-nginx-clusterip
...
curl my-nginx-clusterip
...
curl $POD
...
```
### DNS egress 
Fix the DNS resolving <br>
If required (depending on cluster initialisation) label the `kube-system` namespace
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
...
curl my-nginx-clusterip
...
curl $POD
...
```
### HTTP ingress (server-side)
Enable access on port 80
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
Check connectivity
```
kubectl run -it --rm -n prod-nginx --image xxradar/hackon --env="POD=$POD" debugnslookup my-nginx-clusterip
```
```
nslookup my-nginx-clusterip
...
curl my-nginx-clusterip
...
curl $POD
...
```
### HTTP egress (client-side)
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
...
curl my-nginx-clusterip
...
curl $POD
...
```

### HTTP ingress different namespace (client-side)
Connectivity form a different namespace ...
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
```
kubectl label ns myhackns project=debug
```
```
kubectl run -it --rm  -n myhackns --image xxradar/hackon -l mode=debug debug
```
```
nslookup my-nginx-clusterip.prod-nginx
...
curl my-nginx-clusterip.prod-nginx
...
```
### Additional examples
```
kubectl run -it --rm  -n myhackns --image xxradar/hackon -l mode=nodebug debug
curl my-nginx-clusterip.prod-nginx
...
```
```
kubectl run -it --rm  -n dev-nginx --image xxradar/hackon -l mode=debug debug
curl my-nginx-clusterip.prod-nginx
...
```
Fix access from `dev-nginx` namespace
```
kubectl label ns dev-nginx project=debug
```
```
kubectl run -it --rm  -n dev-nginx --image xxradar/hackon -l mode=debug debug
```
```
curl my-nginx-clusterip.prod-nginx
...
```
## Advanced: Cilium cluster wide network policy example
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
```
kubectl run -it --rm -n myhackns --image xxradar/hackon --env="POD=$POD" debug
```
```
nslookup www.radarhack.com
...
curl https://www.radarhack.com
...
```
In an other terminal 
```
kubectl label po/debug -n myhackns  quarantine=true 
```
Retun to the pod
```
curl https://www.radarhack.com
...
curl https://www.radarhack.com
...
```
## Cleanup
```
kubectl delete ns prod-nginx
kubectl delete ns dev-nginx
kubectl delete ns myhackns
kubectl delete CiliumClusterwideNetworkPolicy quarantine
```

