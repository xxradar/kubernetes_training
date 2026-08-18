# LAB090 - Admission control with Kyverno

An **admission controller** sits in the API request path: after authentication and authorization, but **before** the object is persisted to etcd. Two kinds matter here: **validating** (accept or reject the request) and **mutating** (change the object on the way in). Pod Security Admission (LAB074) is a built-in *validating* controller, but it only offers the three fixed PSS profiles. A **policy engine** lets you write your own rules and, unlike PSA, **mutate** resources.

**Kyverno** is the easiest policy engine to adopt: its policies are plain **Kubernetes YAML**, so there is no new language to learn. (Gatekeeper/OPA does the same job but its policies are written in **Rego**, a separate language.) A Kyverno rule can `validate`, `mutate`, or `generate`.

> Standalone lab. Kyverno installs into namespace `kyverno`; test workloads go in `kyverno-demo`. Any cluster works (kind is fine).

## Install Kyverno
```
kubectl create -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml
kubectl -n kyverno rollout status deploy/kyverno-admission-controller --timeout=180s
```
```
kubectl create ns kyverno-demo
```

## A. Validate in Audit mode
`require-limits.yaml` is a `ClusterPolicy` that requires CPU and memory limits on every container (the rule from LAB072, now enforced centrally). It starts in **Audit**: violations are recorded, not blocked, which is how you roll out a policy safely.
```
kubectl apply -f require-limits.yaml
```
Create a pod with no limits:
```
kubectl run bad -n kyverno-demo --image=busybox --restart=Never --command -- sleep 3600
```
**Expected:** the pod **is created** (Audit does not block). The violation is recorded in a **PolicyReport**, generated asynchronously, so wait ~30s:
```
kubectl get policyreports -n kyverno-demo
```
```
NAME     KIND   NAME   PASS   FAIL   WARN   ERROR   SKIP   AGE
<uid>    Pod    bad    0      1      0      0       0      33s
```
The `FAIL 1` is the `require-limits` policy flagging the pod. Auditing first lets you inventory violations before you start rejecting anything.

## B. Validate in Enforce mode
Flip the rule to **Enforce** so violations are rejected at admission (`failureAction` is per-rule in Kyverno 1.11+):
```
kubectl patch clusterpolicy require-limits --type json \
  -p='[{"op":"replace","path":"/spec/rules/0/validate/failureAction","value":"Enforce"}]'
```
Now a pod without limits is refused:
```
kubectl run bad2 -n kyverno-demo --image=busybox --restart=Never --command -- sleep 3600
```
**Expected:** blocked at the webhook:
```
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request:
require-limits:
  check-resource-limits: 'validation error: CPU and memory limits are required ...'
```
A pod that sets limits is admitted:
```
kubectl apply -n kyverno-demo -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: good
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","sleep 3600"]
    resources:
      limits:
        cpu: 50m
        memory: 32Mi
EOF
```

## C. Mutate: inject a default
Validation **rejects** bad input; mutation **fixes** it. This is the thing PSA cannot do. `add-default-label.yaml` adds a `team=default` label to any pod that does not already have one (`+(team)` means "add if absent", it never overwrites):
```
kubectl apply -f add-default-label.yaml
```
The `good` pod from step B carried no `team` label. Check that Kyverno added it:
```
kubectl get pod good -n kyverno-demo -o jsonpath='{.metadata.labels}{"\n"}'
```
**Expected:** `{"team":"default"}`. The same mechanism can inject a default `securityContext`, add a sidecar, or set `imagePullPolicy`, turning "reject bad input" into "quietly correct it".

## PSA vs Kyverno vs Gatekeeper
- **Pod Security Admission** (LAB074): built in, zero install, but only the three fixed PSS profiles, and validate-only.
- **Kyverno**: custom policies in **plain YAML**; can **validate** (Audit to observe, Enforce to block) **and mutate** (and generate); per-rule action. Easiest to adopt.
- **Gatekeeper (OPA)**: also policy-as-code, but policies are written in **Rego** with a ConstraintTemplate + Constraint model. A more powerful expression language at the cost of a steeper learning curve.

## Explore it yourself
* Roll a new policy out in `Audit`, inventory the PolicyReports across a namespace, then switch to `Enforce`. Why is audit-first the safe pattern?
* Write a validate rule that blocks `privileged: true` (LAB074).
* Write a mutate rule that injects `allowPrivilegeEscalation: false` into every container. Which is a better fit than PSA here?
* Point a policy at Deployments instead of Pods. How does Kyverno's auto-generation handle the pod template?

## Cleanup
```
kubectl delete clusterpolicy require-limits add-default-label
kubectl delete ns kyverno-demo
kubectl delete -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml
```

> Takeaway: admission controllers gate every API write. PSA (LAB074) is the built-in, fixed-profile validator; **Kyverno** adds custom, YAML-defined policies that can **validate** (Audit to observe, Enforce to block) and **mutate** (auto-fix), while **Gatekeeper/OPA** does the same in Rego. The safe rollout is always Audit first, read the PolicyReports, then switch to Enforce.
