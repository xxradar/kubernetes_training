# Kubernetes for Network & Security Engineers

## Session 1 - Foundations
- Intro to k8s + control plane (api-server, etcd, scheduler, controller-manager, kubelet/CRI)
- Setting up a cluster (kind)
- Networking and CNI (Calico / Cilium)
- CoreDNS / service discovery
- Running a first pod
- Namespaces
- Pods, ReplicaSets, Deployments
- Labels and selectors
- Replicas and scaling
- DaemonSets and hostNetwork
- StatefulSets
+ labs 000, 010, 020

## Session 2 - Services & Networking
- kube-proxy dataplane (iptables / IPVS / eBPF, Cilium kube-proxy replacement)
- Services (ClusterIP) + Endpoints / EndpointSlices
- Health probes (readiness / liveness / startup)
- Accessing services (port-forward, kubectl proxy)
- Services (NodePort)
- Services (LoadBalancer, MetalLB)
+ labs 030, 032, 035, 040, 050

## Session 3
- Intro storage
- Secrets and ConfigMaps
- Ingress
- Gateway API
+ labs 061, 062, 063, 065

## Session 4
- Resource requests / limits (as a security control)
- securityContext hardening (runAsNonRoot, readOnlyRootFS, drop caps, seccomp)
- Pod Security Standards
- Image and IaC security
- Network policies
- labs 072, 074, 075

## Session 5
- Introduction to Helm
- RBAC + ServiceAccounts
- Audit logs
- Events
- Admission control (Kyverno / Gatekeeper)
- Observability & metrics (metrics-server, Prometheus model, logs, Hubble)
- lab 090 100 110






