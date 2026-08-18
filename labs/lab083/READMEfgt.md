# LAB110 - Fortinet K8S SDN connector

### K8S SDN connector requirements
- Service Account 
- Token
- URL / IP
- K8S CA 

Note: The SDN connector will verify the certiifcate of the K8S API and fail if not trusted.



### Create a K8S SA
```
kubectl create sa fortigateconnector
```

### Create a service account token (for recent versions of K8S)
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

### Create a K8S role
```
kubectl apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
 name: fgt-connector
rules:
- apiGroups: [""]

  resources: ["pods", "namespaces", "nodes" , "services"]
  verbs: ["get", "watch", "list"]
EOF
```

### Create a K8S rolebinding
```
kubectl create clusterrolebinding fgt-connector --clusterrole=fgt-connector --serviceaccount=default:fortigateconnector
```

### Find the K8S API endpoint
```
kubectl cluster-info
```
```
Kubernetes control plane is running at https://10.1.2.12:6443
CoreDNS is running at https://10.1.2.12:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
or
```
CONTEXT=$(kubectl config view -o jsonpath='{.current-context}')
kubectl config view -o jsonpath='{.clusters[?(@.name=="'$CONTEXT'")].cluster.server}'

## should work if you only have access to a single cluster
kubectl config view -o jsonpath='{.clusters[0].cluster.server}'  
```
```
https://10.1.2.12:6443
```
### Extract the token
Tokens are store in secrets. Tokens are encoded (not encrypted)

To find the token 
```
 SECRET=$(kubectl get sa fortigateconnector -o jsonpath='{.secrets[0].name}')
 kubectl get  secret $SECRET -o jsonpath='{.data.token}' | base64 --decode
```
It should look like this ...
```
eyJhbGciOiJSUzI1NiIsImtpZCI6ImJfMVV5ZFpsbThxMGgyQWV5UW4wYXM2cXFDQ05PYVZsSDA4YU1EOWVMMEUifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImZvcnRpZ2F0ZWNvbm5lY3Rvci10b2tlbi1mcm1yayIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJmb3J0aWdhdGVjb25uZWN0b3IiLCJrdWJlcm5ldGVhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIxNjg3YjUxMC0yZDk5LTQ4NzEtYWRhMS0xMzA5NTE0OTQwZjQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6ZGVmYXVsdDpmb3J0aWdhdGVjb25uZWN0b3IifQ.hyWHnEQ3Mewb_zkCRJBkOVwf1bVO4oHQanqIXXZU38bNsi5j4p3SKzQFpzA4bywGKFYiRZ1TS9f1twTIVZJpOErQedr4m6tPfTMO8Md8r6d-nCewfd_eGlG7Mt9sS2GGU8DxpjwnsAgUtynIEZj1TWRpYQGSgFdKANanBbVCqei7uc8phaGyi82DqBsEZr1HjhlyXTPEh-MvLEjWm2NhC0Zhg75KCF_wged7SZoa9spSCJqIVT20_ykKVdqaF4dydBw2HRR8ssaNN4uLz4eB-Gskt3o-ME3ecE6sHK8OIO7nBYF0pLqrR0217If-NXcENL-sCC1VZ-w
```

### Extract the K8S CA certificate
```
kubectl get  secret $SECRET -o jsonpath='{.data.ca\.crt}' | base64 --decode >ca.crt
cat ca.crt
```
It should look like this ...
```
-----BEGIN CERTIFICATE-----
MIIC/jCCAeagAwIBAgIBADANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwprdWJl
cm5ldGVzMB4XDTIyMDIwMjA5MzAzMVoXDTMyMDEzMTA5MzAzMVowFTETMBEGA1UE
AxMKa3ViZXJuZXRlczCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALVV
tiuf1sNW7fC5C25CVSgRfOvUob9vd5wr3/jRRVnSeEG/HJDWwx5HIHIivXl4/J17
ZhT0hk9V6fKYDBJ63f09dAcJEDZhrvrnmOTrZyMwnhBjmqURkx2ol+yifGgOwNJn
/vyCMNLymJVBA64rIW3vgVUCBZTQjocDI++YBo1VqnLDOrrkMbYg/qAjglI/97PP
oWbQuJf36xhldfAER5pi2XZSPalf13dnlt9uqGsVOWidth3jYXKQe0fWexRhHwHE
F2QB8jh8vG41WFsNVxeBbrORv/6XIsjYjN8s2GRom09BI7iULje0L/yq/ynIHw3v
s1QQ19XFc9KBfHeFnacCAwEAAaNZMFcwDgYDVR0PAQH/BAQDAgKkMA8GA1UdEwEB
/wQFMAMBAf8wHQYDVR0OBBYEFCdasd asdhALIZzJf9HFg8hLJPIMBUGA1UdEQQO
MAyCCmt1YmVybmV0ZXMwDQYJKoZIhvcNAQELBQADggEBAFoZAVzOrsyPommnIMlH
Pu8bmCor57SPnl7u1eDuNg+WlQLn+YPBT6cb9xTYI+w5f+sJuAbEcAOsTrQ/tflZ
EMct2DJ3uWTOlk8qYwLHQR9QIeG4NLIyW9TjzcJ+GGR8zUzv5X4T8vHaK5xzHlTi
aDNafXHLSpkbdp4nzV7yf7IhSa/EAF9LvuSlleOZNtRmC2p+d5ka+GjfI7vTup1f
E7BdwNewwD0ZTaceDNhaU31s5V67HQPP8/BEfen9FEb6HArY/2is16c6JfGeDTMU
/ZaGHsEeNTXt6n0qQbDq+JuqDvnPp4W5CzUiM70JXbkpdfKS5/RTzaDURtkzzjXn
1xE=
-----END CERTIFICATE-----
```


