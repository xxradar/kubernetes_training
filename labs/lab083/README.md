# LAB083 - RBAC and ServiceAccounts (and how an external firewall integrates)

**RBAC** (Role-Based Access Control) decides who may do what in the cluster. It has three moving parts:

- **Subjects** - who is acting: **users** and **groups** (humans, authenticated outside the cluster) and **ServiceAccounts** (identities for workloads and external systems, managed inside the cluster).
- **Role / ClusterRole** - a set of allowed *verbs* (`get`, `list`, `watch`, `create`, ...) on *resources* (`pods`, `services`, ...). A **Role** is namespaced; a **ClusterRole** is cluster-wide.
- **RoleBinding / ClusterRoleBinding** - grants a Role/ClusterRole to a subject. Binding is the step that actually gives access.

A **ServiceAccount (SA)** is the identity a pod (or an outside integration) uses to talk to the API. It authenticates with a **token**, and the client also needs the **API server URL** and the cluster **CA certificate** to trust the connection. Those three (token, URL, CA) are exactly what a `kubeconfig` bundles.

> Runs on the kind cluster from LAB000.

## A. RBAC basics
Create a ServiceAccount and a read-only ClusterRole, then bind them.
```
kubectl create serviceaccount viewer
```
A ClusterRole granting read-only access to a few core resources:
```
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: readonly-core
rules:
- apiGroups: [""]
  resources: ["pods", "namespaces", "nodes", "services"]
  verbs: ["get", "watch", "list"]
EOF
```
Bind the ClusterRole to the ServiceAccount:
```
kubectl create clusterrolebinding viewer-readonly \
  --clusterrole=readonly-core \
  --serviceaccount=default:viewer
```
Test what that identity can and cannot do with `kubectl auth can-i --as`:
```
kubectl auth can-i list pods --as=system:serviceaccount:default:viewer -A
kubectl auth can-i delete pods --as=system:serviceaccount:default:viewer -A
```
**Expected:** `yes` for `list pods`, `no` for `delete pods`. The SA name in RBAC is always `system:serviceaccount:<namespace>:<name>`.

## B. ServiceAccount tokens
Since Kubernetes 1.24 a ServiceAccount **no longer auto-creates a token Secret**. There are two ways to get a token:

**Short-lived token (TokenRequest API)** - the modern default, expires automatically:
```
kubectl create token viewer
```
Good for scripts and CI. It is time-bound, so it is not suitable for a system that needs a stable, long-lived credential.

**Long-lived token (bound Secret)** - for external integrations that need a durable token, create a Secret of type `kubernetes.io/service-account-token` and let the controller fill it in:
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: viewer-token
  annotations:
    kubernetes.io/service-account.name: viewer
type: kubernetes.io/service-account-token
EOF
```
The token and the cluster CA are now in that Secret (base64-encoded, remember LAB062, encoding is not encryption):
```
kubectl get secret viewer-token -o jsonpath='{.data.token}' | base64 -d; echo
kubectl get secret viewer-token -o jsonpath='{.data.ca\.crt}' | base64 -d | head -3
```
Find the API server URL (the third piece a client needs):
```
kubectl config view -o jsonpath='{.clusters[0].cluster.server}{"\n"}'
```
Token + CA + URL is everything an outside client needs to authenticate as `viewer`, with exactly the read-only rights the ClusterRole grants.

## C. Applied example: the FortiGate Kubernetes SDN connector
A FortiGate firewall integrates with Kubernetes through its **Kubernetes SDN connector**. The connector logs in to the API with a **read-only ServiceAccount token** and continuously **watches pods, services, and their labels**, turning them into **dynamic address objects**. Firewall policies can then be written against labels (for example `app=nginx`) instead of pod IPs, and the address objects update automatically as pods come and go. The connector also verifies the API server's certificate against the cluster **CA**, so it needs the CA too.

This is the RBAC + SA + token pattern from Parts A and B, applied to an external system. The goal here is to show **how it integrates**, not to configure the FortiGate itself.

What the connector needs: a ServiceAccount, a long-lived token, the cluster CA, and the API URL/port.
```
kubectl create serviceaccount fortigateconnector
```
Long-lived token Secret for the connector:
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: fortigatek8sconnectortoken
  annotations:
    kubernetes.io/service-account.name: fortigateconnector
type: kubernetes.io/service-account-token
EOF
```
Read-only ClusterRole over the resources the connector maps, and the binding:
```
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fgt-connector
rules:
- apiGroups: [""]
  resources: ["pods", "namespaces", "nodes", "services"]
  verbs: ["get", "watch", "list"]
EOF
kubectl create clusterrolebinding fgt-connector \
  --clusterrole=fgt-connector \
  --serviceaccount=default:fortigateconnector
```
Now hand the FortiGate the three values it asks for.

API server endpoint (IP/port):
```
kubectl cluster-info
kubectl config view -o jsonpath='{.clusters[0].cluster.server}{"\n"}'
```
The token (paste into the connector's `Secret Token` field):
```
kubectl get secret fortigatek8sconnectortoken -o jsonpath='{.data.token}' | base64 -d; echo
```
The cluster CA (so the connector trusts the API certificate):
```
kubectl get secret fortigatek8sconnectortoken -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
cat ca.crt
```
On the FortiGate you then create a **Kubernetes SDN connector** (Security Fabric > External Connectors) and fill in that IP/port, secret token, and CA. From that point the firewall sees your pods and services by label. Fortinet documents this same flow for FortiOS 6.4 through 7.6.

> The old habit of reading the token from `kubectl get sa <name> -o jsonpath='{.secrets[0].name}'` no longer works on Kubernetes 1.24+, because the SA has no auto-mounted Secret. Reference the Secret you created by name instead, as above.

## Explore it yourself
* Try an action outside the grant: `kubectl auth can-i create services --as=system:serviceaccount:default:fortigateconnector`. Why should a firewall connector never have write verbs?
* Scope the connector to one namespace: swap the ClusterRole/ClusterRoleBinding for a namespaced `Role`/`RoleBinding`. What does the connector lose visibility of?
* Decode the token's middle segment (it is a JWT). Which claims identify the ServiceAccount?

## Cleanup
```
kubectl delete clusterrolebinding viewer-readonly fgt-connector
kubectl delete clusterrole readonly-core fgt-connector
kubectl delete secret viewer-token fortigatek8sconnectortoken
kubectl delete serviceaccount viewer fortigateconnector
rm -f ca.crt
```

> Takeaway: RBAC grants access by binding a **Role/ClusterRole** (verbs on resources) to a **subject** (user, group, or ServiceAccount). A ServiceAccount is an in-cluster identity that authenticates with a **token**; combined with the API **URL** and cluster **CA** it is a full set of credentials. External integrations like the FortiGate SDN connector are just another subject: give them a read-only ClusterRole and a long-lived token, and least privilege still applies.
