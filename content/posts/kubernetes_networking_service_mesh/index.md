---
title: "Kubernetes Networking and Service Mesh"
date: 2023-09-23T16:14:26+07:00
tags: ["Kubernetes", "DevOps"]
author: ["hoangph3"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Expose applications with Kubernetes Services and add traffic management with the Istio service mesh."
summary: "Expose applications with Kubernetes Services and add traffic management with the Istio service mesh."
disableHLJS: false
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
---

## Kubernetes Services

### External access by configure Ingress

Firstly, install nginx ingress:

```sh
minikube addons enable ingress
kubectl get all -n ingress-nginx
```

```
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create--1-6rs7v     0/1     Completed   0          14m
pod/ingress-nginx-admission-patch--1-qf2zd      0/1     Completed   1          14m
pod/ingress-nginx-controller-5f66978484-wtpph   1/1     Running     0          14m

NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.104.234.97    <none>        80:31300/TCP,443:30830/TCP   14m
service/ingress-nginx-controller-admission   ClusterIP   10.109.145.221   <none>        443/TCP                      14m

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           14m

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-5f66978484   1         1         1       14m

NAME                                       COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   1/1           4s         14m
job.batch/ingress-nginx-admission-patch    1/1           5s         14m
```

Create kubia application:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia-deployment
  labels:
    app: kubia
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia:latest
        ports:
        - containerPort: 8080
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: kubia-internal-service
spec:
  # type default is ClusterIP
  selector:
    app: kubia
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

```sh
kubectl apply -f kubia.yaml
```

Create ingress for kubia-internal-service:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kubia-ingress
spec:
  rules:
  - host: kubia.example.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: kubia-internal-service
            port:
              number: 8080
```

```sh
kubectl apply -f ingress.yaml
```

Now we can list of ingress:

```sh
kubectl get ingress
```

```
NAME            CLASS   HOSTS               ADDRESS     PORTS   AGE
kubia-ingress   nginx   kubia.example.com   localhost   80      94s
```

The ADDRESS is localhost, this is the url that kubernetes control plane is running. In this case is `minikube ip: 192.168.49.2`.

Edit `/etc/hosts` file:

```
127.0.0.1       localhost
127.0.1.1       jump-windows

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

# k8s
192.168.49.2 kubia.example.com
```

Go to your browser or use `curl`, we can access service through ingress:

```sh
curl kubia.example.com
```

```
You've hit kubia-deployment-7b895464d7-h8448
```

### Configure TLS for Ingress

Firstly, we create the RSA private key:

```sh
openssl genrsa -out tls.key 2048
```

Next we will use the RSA private key to generate the certificate:

```sh
openssl req -new -x509 -key tls.key -days 360 -subj "/CN=kubia" -out tls.crt
```

Perform base64 encode for `tls.key` and `tls.crt`, then fill to yaml file:

```sh
cat tls.crt | base64 | tr -d "\n"
cat tls.key | base64 | tr -d "\n"
```

Create `tls-secret.yaml` save secret TLS:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
type: kubernetes.io/tls
data:
  # the data is abbreviated in this example
  tls.crt: |
    LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURBVENDQWVtZ0F3SUJBZ0lVUWw2eitJOUY4MmgwQThjbzUxVmJ5K2Vjay9nd0RRWUpLb1pJaHZjTkFRRUwKQlFBd0VERU9NQXdHQTFVRUF3d0ZhM1ZpYVdFd0hoY05Nakl3TkRFd01UWXlOek0wV2hjTk1qTXdOREExTVRZeQpOek0wV2pBUU1RNHdEQVlEVlFRRERBVnJkV0pwWVRDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDCkFRb0NnZ0VCQU1DNURMa3JXMHhJWkVEUFE4N1RYdlMxU3M3UkJKWk9wZGlTcnlaSTFHb0s4cndZRTZQZ2pwM1MKaUNlQ3FHbWRWNGV1aGhmbjk5WGlEQVFYSjRlS1R5TDlndTdvV2M5ZFg4d25KR0tXSjVnRE8zTzFnUWNjcGRhSwppSWRyUTlyMXBPTkIrSjZnR0t6NUtucXpZK2wvL0NSSDU2M0FuZ1h2TWlvODVWTWNOd3ZOV09nVjBpNlRjSk1kCmE4aWk2YXhLQWcwMjlaWjFRRFFPTVhYR3ZNeFFzRkhUM045ZUthbDFKYlZRL2ZFUy9hOHNNbXFYN01ncE0wdkUKV2xLSzF1TjlzeWYyamtKQzlhd0Fac3hveDQxbjd0WENPdTJ5alVqTlQvWGNyU0hPbUtScXRqL215T096N1Myawp0VE5QckVBVWk1U3FNbjBTbG43UXVjVEdtUVIrdDNrQ0F3RUFBYU5UTUZFd0hRWURWUjBPQkJZRUZMZTYvbERqCmZSNk8yOXptTHlLc3hZQmxmb0FMTUI4R0ExVWRJd1FZTUJhQUZMZTYvbERqZlI2TzI5em1MeUtzeFlCbGZvQUwKTUE4R0ExVWRFd0VCL3dRRk1BTUJBZjh3RFFZSktvWklodmNOQVFFTEJRQURnZ0VCQUgyTDh6S0dLUmZrdkMxNgplbW9ESE1jT0MxOEpiWmIzNG5yRDB0MWlleHp4MHJLNTFVRU5TZlJYK2E4Y3Vma2xFREROc0hUNnRHV2xEYmFECkhPeXRUZDNKcXg5bFJNMG1LK1pXZk1ITGNXQVF2bXZWcUNGa3NDcUhHc3ZTb1Q3WVY0OHBqcytpL1VsZnJPcE8KcmlNYk5tbTdHNkdqLzA4VkluTDdIVUZaQnNLdEovNkJ5WXJrR01LUHoyNjB5aHQ2TDJxaUx1NlRIbUN3eGV1YwpNd1J3bWN1ajlzQmFtL3JQOWJoTHJYcThuSUxCSUlEZmlIWE5zYXBnL1I0WGVlcDZaaG8yUDJlSmlHRE1udHRDCkVyVUtnT29IOWdHZmlrR2FiWHNQK0c2dkFFZk1FQXR3L09wVmZHTHNSZnZ1U1RQT0JTVXdkaThvcldUTDFvOVAKMVFjSVRVTT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
  tls.key: |
    LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcFFJQkFBS0NBUUVBd0xrTXVTdGJURWhrUU05RHp0TmU5TFZLenRFRWxrNmwySkt2SmtqVWFncnl2QmdUCm8rQ09uZEtJSjRLb2FaMVhoNjZHRitmMzFlSU1CQmNuaDRwUEl2MkM3dWhaejExZnpDY2tZcFlubUFNN2M3V0IKQnh5bDFvcUloMnREMnZXazQwSDRucUFZclBrcWVyTmo2WC84SkVmbnJjQ2VCZTh5S2p6bFV4dzNDODFZNkJYUwpMcE53a3gxcnlLTHByRW9DRFRiMWxuVkFOQTR4ZGNhOHpGQ3dVZFBjMzE0cHFYVWx0VkQ5OFJMOXJ5d3lhcGZzCnlDa3pTOFJhVW9yVzQzMnpKL2FPUWtMMXJBQm16R2pIaldmdTFjSTY3YktOU00xUDlkeXRJYzZZcEdxMlArYkkKNDdQdExhUzFNMCtzUUJTTGxLb3lmUktXZnRDNXhNYVpCSDYzZVFJREFRQUJBb0lCQURzdkdPY3NsMmIvdkRuaQo3TEh4VzNITzB1QmNkQW9zc09XbmRqNU5rMTNWYXVHMGl5T0NiSW12QTcwT2RPV3FPaDBpelc4OS8zQWhjUXM0CmlSMG9ybERTaFlrVXRhL212dXFWQXFsNzcwRFJqVXBsYlBCZ0xkV0t5WTY4dENQajEvVXFaMDFmWVBTTnVDdmkKTjBhWDFUalhGQ0RaekMyS1hWOTNQLzJiNXBPcXZMRHRiWDhDRkYrcmloYnhGOEk5ZkhmdXBTYzFwQXNYWkNBQQoxRnNXZG83Nmk2YVBuL3ZuRU9kRG1KK09ZN3p5K01hM3E3alF3ajhFMmRkUFdDQkp1UE9yNzEzRGt1TXJUc3RTCjFMRVptWlIzckFJcytxbDI4U0ZVbkdiWHZHZjhSNDBkWnhsNkpzR09hS3d1VjNwck9zN2hUbW9leUhSYlk1V0cKNkV5TXp6a0NnWUVBKzFlSjdoTVR6eFhDVTAxTU5sMjFmYjU1cEpaNm55ZmFPMWI1bUhUR295dndrWEN0NFRUcwovb1plN0tzVUtJZFVmWDZ1OCs0Nms0VzNkZjBwVDZWR2ZHODdwWlBNVVJaMk53SmUzc3FMMnBBNHRzZXQ2N3Q0ClVlaGswdlJYRXRQQXV3SVBlbXhHMXI0Q0FMNFZsMFJOK29UQ1B4YmJrRlRVZ1FsdzUrS29TWU1DZ1lFQXhFdG0KVml4eVluUUZROW0wZk4vWndUZTZVMzZ2S0s3WXF3NHRVelAyejRwQ1RkWEhsREgrWEhtMDR1aFd3dVg0V1hHMwpvNHV5NTFBZmpjS1VJcFVyckgxa3RMWkM4T3RZYzlVY0dUTUZEYzhmNzI0R2dNRXU1c3dYWTJHS1RvbkJOcm8wCktLYmhlTEFpeW9HMGM0OEppS3BtYTYrRVdvSDFNM1EvOS9hWjlsTUNnWUVBMURCeklhcTVibnJRTThOdU0vZW8KNFIrTlVvWTN2MlhGdDVNVjVMK3hjdEFGcU1PWUNDakdhNXJGU01pbG5CR2tJczV3cFQ3WjlQRk9rUzNKVXBRVgpqYmZhZzA3amp4R0hlNmxrcm5JUTM5UWlEUzFHaDEwZGx3aTdGZDF5SlZMZnd3RmFUK0JaYmJHN3Z5UzYxWm0wCnUycVpFdW9aTXlCcXh3VlJiSExONEVFQ2dZRUFuVmhuTnNvNEFrMUg3eVJ5ZGVxbHpTalRsWndsNGJHT0FrZkIKODBEakpXZUpVSVQ5andBb0NZNlJmWldKL242Qy9ZZVhFV1NveXB4Q1BzcnJIWEYvYWF1MTd0bHVmVm5aTkRodQpacENzQzI2dEJhcW5VY3dJd1g1MWZQY3grMVNXNlR5SEZOTDRSMXJBK0p6UnZoTzVLN0NUbXR3OWRxTlhucUFmCnFxOGtxUHNDZ1lFQWcxQng5TksyVVVkZTgyaDZmZmFlWXROVDdXa1RXM2dvandFV2p3WXhzZVBVQ0Q1MWloVUEKYVQ0QVhvaWxsRys1Wnd3Q3Z4V0NYNE1kWTl5SDlVMng5SGhjS1QreCszZnp3NjNPN0g1ampjcHRXYVJhdlZadApsdUlEM1crVnpaL3ZvYnV2WUdScVc5NUhLbThRN2FuU283SzB2TkswUXZzYXJySjZ4c01rcW1FPQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
```

```sh
kubectl apply -f tls-secret.yaml
```

Now create Ingress with TLS:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-kubia-ingress
spec:
  tls:
  - hosts: 
    - kubia.example.com
    secretName: secret-tls
  rules:
  - host: kubia.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubia-internal-service
            port:
              number: 8080
```

```sh
kubectl apply -f tls-ingress.yaml
kubectl get ingress
```

```
NAME                CLASS   HOSTS               ADDRESS     PORTS     AGE
tls-kubia-ingress   nginx   kubia.example.com   localhost   80, 443   17s
```

We can see the PORTS is 80, 443. Let’s use HTTPS to access your service through the Ingress:

```sh
curl -k -v https://kubia.example.com/kubia
```

```
*   Trying 192.168.49.2:443...
* Connected to kubia.example.com (192.168.49.2) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate
*  start date: Apr 10 14:38:48 2022 GMT
*  expire date: Apr 10 14:38:48 2023 GMT
*  issuer: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
* Using HTTP2, server supports multiplexing
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x560b8745fb10)
> GET /kubia HTTP/2
> Host: kubia.example.com
> user-agent: curl/7.81.0
> accept: */*
> 
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
< HTTP/2 200 
< date: Sun, 10 Apr 2022 16:51:41 GMT
< 
You've hit kubia-deployment-7b895464d7-h8448
* Connection #0 to host kubia.example.com left intact
```

### Configure headless service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubia-headless
spec:
  selector:
    app: kubia
  clusterIP: None
  ports:
  - port: 80
    targetPort: 8080
```

A headless service is a service with a service IP but instead of load-balancing it will return the IPs of our associated Pods. This allows us to interact directly with the Pods instead of a proxy.

Let’s list of service:

```sh
kubectl get service
```

```
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
kubernetes               ClusterIP   10.96.0.1      <none>        443/TCP    73d
kubia-headless           ClusterIP   None           <none>        80/TCP     33m
kubia-internal-service   ClusterIP   10.96.220.32   <none>        8080/TCP   3h1m
```

Because headless service is connected to Pod’s IPs without proxy. If we exec to the pod, we can interact with headless service:

```sh
kubectl get pods
```

```
NAME                                READY   STATUS    RESTARTS   AGE
kubia-deployment-7b895464d7-h8448   1/1     Running   0          3h5m
```

```sh
kubectl exec -it kubia-deployment-7b895464d7-h8448 -- bash
```

```
root@kubia-deployment-7b895464d7-h8448:/# curl kubia-headless:8080
You've hit kubia-deployment-7b895464d7-h8448

root@kubia-deployment-7b895464d7-h8448:/# curl kubia-internal-service:8080
```

## Istio Service Mesh

### Configure Istio for Kubernetes cluster

Firstly, starting your minikube cluster:

```
$ minikube start --memory=8192 --cpus=4
...
Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

To build service mesh for kubernetes cluster, we can use istio, let’s download istio from [https://github.com/istio/istio/releases/](https://github.com/istio/istio/releases/) and extracting:

```
$ wget https://github.com/istio/istio/releases/download/1.12.7/istio-1.12.7-linux-amd64.tar.gz

$ tar zvxf istio-1.12.7-linux-amd64.tar.gz
```

Checking the binary istio:

```
$ cd istio-1.12.7 && ls
bin  LICENSE  manifests  manifest.yaml  README.md  samples  tools

$ export PATH=$PWD/bin:$PATH

$ istioctl
Istio configuration command line utility for service operators to
debug and diagnose their Istio mesh.

Usage:
  istioctl [command]

Available Commands:
  ...

Flags:
      ...

Additional help topics:
  istioctl options                           Displays istioctl global options

Use "istioctl [command] --help" for more information about a command.
```

If you have not installed istio:

```
$ kubectl get ns
NAME              STATUS   AGE
default           Active   8m47s
kube-node-lease   Active   8m50s
kube-public       Active   8m50s
kube-system       Active   8m50s

$ kubectl get pods
No resources found in default namespace.
```

Let’s install istio service mesh:

```
$ istioctl install
This will install the Istio 1.12.7 default profile with ["Istio core" "Istiod" "Ingress gateways"] components into the cluster. Proceed? (y/N) y
✔ Istio core installed
✔ Istiod installed- Processing resources for Ingress gateways. Waiting for Deployment/istio-system/istio-ingressgateway
✔ Ingress gateways installed
✔ Installation complete
Making this installation the default for injection and validation.
```

Now listing pods and namespace in kubernetes cluster:

```
$ kubectl get ns
NAME              STATUS   AGE
default           Active   10m
istio-system      Active   62s
kube-node-lease   Active   10m
kube-public       Active   10m
kube-system       Active   10m

$ kubectl get pods -n istio-system
NAME                                    READY   STATUS    RESTARTS   AGE
istio-ingressgateway-5744ff657c-cb6z2   1/1     Running   0          42s
istiod-76f7bb65df-tcfvj                 1/1     Running   0          66s

$ kubectl get svc -A              
NAMESPACE      NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                      AGE
default        kubernetes             ClusterIP      10.96.0.1        <none>        443/TCP                                      10m
istio-system   istio-ingressgateway   LoadBalancer   10.111.182.40    <pending>     15021:31549/TCP,80:30496/TCP,443:32456/TCP   61s
istio-system   istiod                 ClusterIP      10.103.110.234   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        86s
kube-system    kube-dns               ClusterIP      10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP                       10m
```

We can get a list of the ports available with the istio-ingressgateway service using:

```
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.111.182.40   <pending>     15021:31549/TCP,80:30496/TCP,443:32456/TCP   10m

$ kubectl describe svc istio-ingressgateway -n istio-system
Name:                     istio-ingressgateway
Namespace:                istio-system
Labels:                   app=istio-ingressgateway
                          install.operator.istio.io/owning-resource=unknown
                          install.operator.istio.io/owning-resource-namespace=istio-system
                          istio=ingressgateway
                          istio.io/rev=default
                          operator.istio.io/component=IngressGateways
                          operator.istio.io/managed=Reconcile
                          operator.istio.io/version=1.12.7
                          release=istio
Annotations:              <none>
Selector:                 app=istio-ingressgateway,istio=ingressgateway
Type:                     LoadBalancer
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.111.182.40
IPs:                      10.111.182.40
Port:                     status-port  15021/TCP
TargetPort:               15021/TCP
NodePort:                 status-port  31549/TCP
Endpoints:                172.17.0.4:15021
Port:                     http2  80/TCP
TargetPort:               8080/TCP
NodePort:                 http2  30496/TCP
Endpoints:                172.17.0.4:8080
Port:                     https  443/TCP
TargetPort:               8443/TCP
NodePort:                 https  32456/TCP
Endpoints:                172.17.0.4:8443
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

The output here shows that the istio-ingressgateway service is forwarding requests from port `80` to port `30496` (http2).

The load balancer listener is set to listen on HTTP port 80, which is the port for the NGINX web server application used in the virtual service in this example.

Step 1: Create the namespace `my-namespace` and enable automatic proxy sidecar injection.

```
$ kubectl create ns my-namespace
namespace/my-namespace created

$ kubectl label ns my-namespace istio-injection=enabled
namespace/my-namespace labeled

$ kubectl get ns --show-labels
NAME              STATUS   AGE   LABELS
default           Active   60m   kubernetes.io/metadata.name=default
istio-system      Active   51m   kubernetes.io/metadata.name=istio-system
kube-node-lease   Active   60m   kubernetes.io/metadata.name=kube-node-lease
kube-public       Active   60m   kubernetes.io/metadata.name=kube-public
kube-system       Active   60m   kubernetes.io/metadata.name=kube-system
my-namespace      Active   30m   istio-injection=enabled,kubernetes.io/metadata.name=my-namespace
```

Step 2: Create the NGINX deployment and NGINX service by create the manifest file `nginx.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: webserver
  name: my-nginx
  namespace: my-namespace
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webserver
  template:
    metadata:
      labels:
        app: webserver
    spec:
      containers:
      - image: nginx
        name: my-nginx
        ports:
        - containerPort: 80 # matched targetPort
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: my-nginx
  name: webserver
  namespace: my-namespace
spec:
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 80 # matched containerPort
  selector:
    app: webserver
  type: ClusterIP
```

```
$ kubectl apply -f nginx.yaml
deployment.apps/my-nginx created
service/webserver created

$ kubectl get deploy,svc,po -n my-namespace
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-nginx   3/3     3            3           19s

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/webserver   ClusterIP   10.96.221.204   <none>        80/TCP    19s

NAME                            READY   STATUS    RESTARTS   AGE
pod/my-nginx-856fb4777f-7s67q   2/2     Running   0          19s
pod/my-nginx-856fb4777f-8bhj2   2/2     Running   0          19s
pod/my-nginx-856fb4777f-d77fs   2/2     Running   0          19s
```

We can see `2/2` in READY column, means that the pods have deployed with sidecar proxy. To validate this, we describe one of the pods:

```
$ kubectl describe pods my-nginx-856fb4777f-7s67q -n my-namespace
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  46s                default-scheduler  Successfully assigned my-namespace/my-nginx-856fb4777f-7s67q to minikube
  Normal   Pulled     45s                kubelet            Container image "docker.io/istio/proxyv2:1.12.7" already present on machine
  Normal   Created    45s                kubelet            Created container istio-init
  Normal   Started    45s                kubelet            Started container istio-init
  Normal   Pulling    45s                kubelet            Pulling image "nginx"
  Normal   Pulled     41s                kubelet            Successfully pulled image "nginx" in 3.441922626s
  Normal   Created    41s                kubelet            Created container my-nginx
  Normal   Started    41s                kubelet            Started container my-nginx
  Normal   Pulled     41s                kubelet            Container image "docker.io/istio/proxyv2:1.12.7" already present on machine
  Normal   Created    41s                kubelet            Created container istio-proxy
  Normal   Started    41s                kubelet            Started container istio-proxy
  Warning  Unhealthy  38s (x3 over 40s)  kubelet            Readiness probe failed: Get "http://172.17.0.5:15021/healthz/ready": dial tcp 172.17.0.5:15021: connect: connection refused
```

Because my environment does not provide an external load balancer for the ingress gateway, the connection refused to `172.17.0.5:15021`. But don’t worry about it, we can access the gateway using the service’s node port (`30496`).

Step 3: Create an ingress gateway for the NGINX service by create the manifest file `nginx-gateway.yaml`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: my-nginx-gateway
  namespace: my-namespace
spec:
  selector:
    istio: ingressgateway # get from labels when describe istio-ingressgateway service
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
      - "mynginx.example.com"
```

```
$ kubectl apply -f nginx-gateway.yaml
gateway.networking.istio.io/my-nginx-gateway created

$ kubectl get gateways.networking.istio.io -n my-namespace 
NAME               AGE
my-nginx-gateway   14m
```

Step 4: Create a virtual service for the ingress gateway by create the manifest file `nginx-virtualservice.yaml`.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-nginx-virtualservice
  namespace: my-namespace
spec:
  hosts:
  - "mynginx.example.com"
  gateways:
  - my-nginx-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        port:
          number: 80
        host: webserver # matched .metadata.name in Service
```

```
$ kubectl apply -f nginx-virtualservice.yaml
virtualservice.networking.istio.io/my-nginx-virtualservice created

$ kubectl get virtualservices.networking.istio.io -n my-namespace 
NAME                      GATEWAYS               HOSTS                     AGE
my-nginx-virtualservice   ["my-nginx-gateway"]   ["mynginx.example.com"]   17s
```

To confirm the ingress gateway is serving the application to the load balancer, use:

```
$ minikube ip
192.168.49.2

$ kubectl describe svc istio-ingressgateway -n istio-system | grep http2
Port:                     http2  80/TCP
NodePort:                 http2  30496/TCP

$ curl -I -HHost:mynginx.example.com 192.168.49.2:30496
HTTP/1.1 200 OK
server: istio-envoy
date: Sun, 22 May 2022 13:07:54 GMT
content-type: text/html
content-length: 615
last-modified: Tue, 25 Jan 2022 15:03:52 GMT
etag: "61f01158-267"
accept-ranges: bytes
x-envoy-upstream-service-time: 17
```

To monitoring and data visualization, we can use `kiali` - an observability console for Istio with service mesh configuration and validation capabilities.

Istio provides a basic sample installation to quickly get Kiali up and running:

```
$ kubectl apply -f prometheus.yaml
serviceaccount/prometheus created
configmap/prometheus created
clusterrole.rbac.authorization.k8s.io/prometheus created
clusterrolebinding.rbac.authorization.k8s.io/prometheus created
service/prometheus created
deployment.apps/prometheus created

$ kubectl apply -f kiali.yaml
serviceaccount/kiali created
configmap/kiali created
clusterrole.rbac.authorization.k8s.io/kiali-viewer created
clusterrole.rbac.authorization.k8s.io/kiali created
clusterrolebinding.rbac.authorization.k8s.io/kiali created
role.rbac.authorization.k8s.io/kiali-controlplane created
rolebinding.rbac.authorization.k8s.io/kiali-controlplane created
service/kiali created
deployment.apps/kiali created

$ kubectl get svc -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.111.182.40    <pending>     15021:31549/TCP,80:30496/TCP,443:32456/TCP   4h34m
istiod                 ClusterIP      10.103.110.234   <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        4h34m
kiali                  ClusterIP      10.100.112.235   <none>        20001/TCP,9090/TCP                           8s
prometheus             ClusterIP      10.110.80.171    <none>        9090/TCP                                     54s
```

Instead expose kiali service, we can forward port from kiali service to localhost by command line:

```
$ kubectl port-forward svc/kiali -n istio-system 20001
Forwarding from 127.0.0.1:20001 -> 20001
Forwarding from [::1]:20001 -> 20001
```

Then, access Kiali by visiting http://127.0.0.1:20001 in your preferred web browser.

You can get full source code here: [service](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/service), [istio-service-mesh](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/istio-service-mesh).
