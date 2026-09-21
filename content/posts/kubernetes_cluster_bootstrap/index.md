---
title: "Bootstrapping Kubernetes Clusters"
date: 2023-09-23T16:14:26+07:00
tags: ["Kubernetes", "DevOps"]
author: ["hoangph3"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Stand up Kubernetes clusters with kubeadm using stacked and external etcd topologies, plus certificate rotation and multi-cluster contexts."
summary: "Stand up Kubernetes clusters with kubeadm using stacked and external etcd topologies, plus certificate rotation and multi-cluster contexts."
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

## Bootstrapping a Basic Cluster

### Create kubernetes cluster using kubeadm and ansible from scratch

### Starting Virtual Machine

Step 1: Create master and worker server (–provision flag to run script when startup)

```sh
vagrant up --provision
```

Step 2: Checking connection

```sh
sshpass -p vagrant ssh vagrant@192.168.56.10
sshpass -p vagrant ssh vagrant@192.168.56.11
sshpass -p vagrant ssh vagrant@192.168.56.12
```

### Create cluster kubernetes

Step 1: Build environment by apt dependencies and config for all server

```sh
ansible-playbook -i hosts build-env.yml
```

Step 2: Build master node

```sh
ansible-playbook -i hosts build-master.yml
```

Step 3: Build worker node

```sh
ansible-playbook -i hosts build-worker.yml
```

Step 4: Explore cluster

```sh
sshpass -p vagrant ssh vagrant@192.168.56.10
kubectl get nodes
kubectl get po -n kube-system
```

```
NAME       STATUS   ROLES                  AGE     VERSION
master     Ready    control-plane,master   34m     v1.23.0
worker-1   Ready    <none>                 5m36s   v1.23.0
worker-2   Ready    <none>                 5m48s   v1.23.0

NAME                             READY   STATUS    RESTARTS   AGE
coredns-64897985d-92fhv          1/1     Running   0          34m
coredns-64897985d-kf6bt          1/1     Running   0          34m
etcd-master                      1/1     Running   2          35m
kube-apiserver-master            1/1     Running   2          35m
kube-controller-manager-master   1/1     Running   2          35m
kube-flannel-ds-6pcq5            1/1     Running   0          6m22s
kube-flannel-ds-wfq5l            1/1     Running   0          34m
kube-flannel-ds-xzh2s            1/1     Running   0          6m10s
kube-proxy-9vr5m                 1/1     Running   0          34m
kube-proxy-cbm87                 1/1     Running   0          6m10s
kube-proxy-nj2l5                 1/1     Running   0          6m22s
kube-scheduler-master            1/1     Running   2          35m
```

Step 5: Deploy application

- Access to master node:

```sh
sshpass -p vagrant ssh vagrant@192.168.56.10
```

- Create file `demo-nginx.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector: 
    app: nginx
  type: NodePort  
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32000
```

- Create pods:

```sh
kubectl apply -f demo-nginx.yaml
```

- Get pods status:

```sh
kubectl get pods
```

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-74d589986c-75xqx   1/1     Running   0          47s
nginx-deployment-74d589986c-lrgq5   1/1     Running   0          47s
```

- Access application from external service: `curl <workerIP>:<nodePort>`

```sh
curl http://192.168.56.11:32000
curl http://192.168.56.12:32000
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Stacked etcd Topology

### Create kubernetes cluster using kubeadm and ansible from scratch

### Starting Virtual Machine

Step 1: Create master and worker server (–provision flag to run script when startup)

```sh
vagrant up --provision
```

Step 2: Checking connection

```sh
sshpass -p vagrant ssh vagrant@192.168.56.10
sshpass -p vagrant ssh vagrant@192.168.56.11
sshpass -p vagrant ssh vagrant@192.168.56.12
sshpass -p vagrant ssh vagrant@192.168.56.13
sshpass -p vagrant ssh vagrant@192.168.56.14
```

### Create cluster kubernetes

Step 1: Build environment by apt dependencies and config for master and worker node

```sh
ansible-playbook -i hosts build-env-cluster.yml
```

Step 2: Build load balancer

```sh
ansible-playbook -i hosts build-load-balancer.yml
```

Step 3: Build master and worker node

```sh
ansible-playbook -i hosts build-master-and-worker.yml
```

Step 4: Explore cluster

```sh
sshpass -p vagrant ssh vagrant@192.168.56.10
kubectl get nodes
kubectl get po -n kube-system -o wide
```

```
NAME       STATUS   ROLES                  AGE   VERSION
master-1   Ready    control-plane,master   32m   v1.23.0
master-2   Ready    control-plane,master   31m   v1.23.0
worker-1   Ready    <none>                 30m   v1.23.0
worker-2   Ready    <none>                 30m   v1.23.0

NAME                               READY   STATUS    RESTARTS      AGE   IP              NODE       NOMINATED NODE   READINESS GATES
coredns-64897985d-2nwxr            1/1     Running   0             32m   10.244.0.3      master-1   <none>           <none>
coredns-64897985d-gnh4h            1/1     Running   0             32m   10.244.0.2      master-1   <none>           <none>
etcd-master-1                      1/1     Running   3             33m   192.168.56.10   master-1   <none>           <none>
etcd-master-2                      1/1     Running   0             31m   192.168.56.11   master-2   <none>           <none>
kube-apiserver-master-1            1/1     Running   3             33m   192.168.56.10   master-1   <none>           <none>
kube-apiserver-master-2            1/1     Running   0             31m   192.168.56.11   master-2   <none>           <none>
kube-controller-manager-master-1   1/1     Running   4 (31m ago)   33m   192.168.56.10   master-1   <none>           <none>
kube-controller-manager-master-2   1/1     Running   0             31m   192.168.56.11   master-2   <none>           <none>
kube-flannel-ds-2mjsv              1/1     Running   0             30m   192.168.56.12   worker-1   <none>           <none>
kube-flannel-ds-cmmf8              1/1     Running   0             32m   192.168.56.10   master-1   <none>           <none>
kube-flannel-ds-kqrd6              1/1     Running   0             30m   192.168.56.13   worker-2   <none>           <none>
kube-flannel-ds-r4kcz              1/1     Running   0             31m   192.168.56.11   master-2   <none>           <none>
kube-proxy-h4fh5                   1/1     Running   0             32m   192.168.56.10   master-1   <none>           <none>
kube-proxy-mdfm7                   1/1     Running   0             30m   192.168.56.12   worker-1   <none>           <none>
kube-proxy-wsr69                   1/1     Running   0             31m   192.168.56.11   master-2   <none>           <none>
kube-proxy-zkhpf                   1/1     Running   0             30m   192.168.56.13   worker-2   <none>           <none>
kube-scheduler-master-1            1/1     Running   4 (31m ago)   33m   192.168.56.10   master-1   <none>           <none>
kube-scheduler-master-2            1/1     Running   0             31m   192.168.56.11   master-2   <none>           <none>
```

Step 5: Deploy application

- Access to master node:

```sh
sshpass -p vagrant ssh vagrant@192.168.56.10
```

- Create file `demo-nginx.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 4
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector: 
    app: nginx
  type: NodePort  
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32000
```

- Create pods:

```sh
kubectl apply -f demo-nginx.yaml
```

- Get pods status:

```sh
kubectl get pods -o wide
```

```
NAME                               READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-8d545c96d-6r4vr   1/1     Running   0          41s   10.244.3.2   worker-1   <none>           <none>
nginx-deployment-8d545c96d-jftnn   1/1     Running   0          41s   10.244.2.3   worker-2   <none>           <none>
nginx-deployment-8d545c96d-vll4h   1/1     Running   0          41s   10.244.2.2   worker-2   <none>           <none>
nginx-deployment-8d545c96d-x5m5g   1/1     Running   0          41s   10.244.3.3   worker-1   <none>           <none>
```

- Access application from external service: `curl <workerIP>:<nodePort>`

```sh
curl http://192.168.56.12:32000
curl http://192.168.56.13:32000
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## External etcd Topology

### Create kubernetes cluster using kubeadm and ansible from scratch

### Starting Virtual Machine

Step 1: Create master and worker server (–provision flag to run script when startup)

```sh
vagrant up --provision
```

Step 2: Checking connection

```sh
sshpass -p vagrant ssh vagrant@192.168.56.11
sshpass -p vagrant ssh vagrant@192.168.56.12
sshpass -p vagrant ssh vagrant@192.168.56.21
sshpass -p vagrant ssh vagrant@192.168.56.22
sshpass -p vagrant ssh vagrant@192.168.56.15
```

### Generate the certificate

Firstly, install `cfssl`:

```sh
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64

cd Downloads
chmod +x cfssl*

sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
sudo mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
```

```sh
cd certs
chmod +x gen.sh
./gen.sh
```

```
2022/04/04 13:33:36 [INFO] generating a new CA key and certificate from CSR
2022/04/04 13:33:36 [INFO] generate received request
2022/04/04 13:33:36 [INFO] received CSR
2022/04/04 13:33:36 [INFO] generating key: rsa-2048
2022/04/04 13:33:37 [INFO] encoded CSR
2022/04/04 13:33:37 [INFO] signed certificate with serial number 168845797640225205009115083971177470265899005809
2022/04/04 13:33:37 [INFO] generate received request
2022/04/04 13:33:37 [INFO] received CSR
2022/04/04 13:33:37 [INFO] generating key: rsa-2048
2022/04/04 13:33:37 [INFO] encoded CSR
2022/04/04 13:33:37 [INFO] signed certificate with serial number 445423558377599319684907386638393716430750399638
2022/04/04 13:33:37 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

Now we will verify the ca certificate and private key were generated:

```sh
ls -la
```

```
total 40
drwxr-xr-x 3 ph3 ph3 4096 Apr  4 13:33 .
drwxr-xr-x 4 ph3 ph3 4096 Apr  4 13:29 ..
-rw-r--r-- 1 ph3 ph3  997 Apr  4 13:33 ca.csr
-rw------- 1 ph3 ph3 1679 Apr  4 13:33 ca-key.pem
-rw-r--r-- 1 ph3 ph3 1350 Apr  4 13:33 ca.pem
drwxr-xr-x 2 ph3 ph3 4096 Apr  4 12:38 config
-rwxr-xr-x 1 ph3 ph3  235 Apr  4 13:06 gen.sh
-rw-r--r-- 1 ph3 ph3 1249 Apr  4 13:33 server.csr
-rw------- 1 ph3 ph3 1679 Apr  4 13:33 server-key.pem
-rw-r--r-- 1 ph3 ph3 1610 Apr  4 13:33 server.pem
```

### Create cluster kubernetes

Step 1: Build environment by apt dependencies and config for master and worker node

```sh
ansible-playbook -i hosts build-env-cluster.yml
```

Step 2: Build load balancer

```sh
ansible-playbook -i hosts build-load-balancer.yml
```

Step 3: Build external etcd cluster on master nodes

```sh
ansible-playbook -i hosts build-etcd.yml
```

After finish, you can see the etcd cluster status:

```
TASK [print etcd status cluster] ***************************************************************************************************************************************
ok: [master-1] => {
    "msg": [
        "+------------------+---------+----------+----------------------------+----------------------------+------------+",
        "|        ID        | STATUS  |   NAME   |         PEER ADDRS         |        CLIENT ADDRS        | IS LEARNER |",
        "+------------------+---------+----------+----------------------------+----------------------------+------------+",
        "| 7de60f185c634ebb | started | master-2 | https://192.168.56.12:2380 | https://192.168.56.12:2379 |      false |",
        "| e81c9fc39b7ba9f8 | started | master-1 | https://192.168.56.11:2380 | https://192.168.56.11:2379 |      false |",
        "+------------------+---------+----------+----------------------------+----------------------------+------------+"
    ]
}
ok: [master-2] => {
    "msg": [
        "+------------------+---------+----------+----------------------------+----------------------------+------------+",
        "|        ID        | STATUS  |   NAME   |         PEER ADDRS         |        CLIENT ADDRS        | IS LEARNER |",
        "+------------------+---------+----------+----------------------------+----------------------------+------------+",
        "| 7de60f185c634ebb | started | master-2 | https://192.168.56.12:2380 | https://192.168.56.12:2379 |      false |",
        "| e81c9fc39b7ba9f8 | started | master-1 | https://192.168.56.11:2380 | https://192.168.56.11:2379 |      false |",
        "+------------------+---------+----------+----------------------------+----------------------------+------------+"
    ]
}
```

Step 4: Build master and worker node

```sh
ansible-playbook -i hosts build-master-and-worker.yml
```

Step 5: Verify the cluster

```sh
sshpass -p vagrant ssh vagrant@192.168.56.11
kubectl get nodes
kubectl get po -n kube-system -o wide
```

```
NAME                               READY   STATUS    RESTARTS         AGE     IP              NODE       NOMINATED NODE   READINESS GATES
coredns-64897985d-fppdp            0/1     Running   3                30m     10.244.0.3      master-1   <none>           <none>
coredns-64897985d-t4mxq            0/1     Running   3                30m     10.244.0.2      master-1   <none>           <none>
kube-apiserver-master-1            1/1     Running   4                30m     192.168.56.11   master-1   <none>           <none>
kube-apiserver-master-2            1/1     Running   0                3m11s   192.168.56.12   master-2   <none>           <none>
kube-controller-manager-master-1   1/1     Running   4                30m     192.168.56.11   master-1   <none>           <none>
kube-controller-manager-master-2   1/1     Running   1                25m     192.168.56.12   master-2   <none>           <none>
kube-flannel-ds-4qtvn              1/1     Running   1                25m     192.168.56.12   master-2   <none>           <none>
kube-flannel-ds-fxptk              1/1     Running   12 (3m37s ago)   28m     192.168.56.11   master-1   <none>           <none>
kube-flannel-ds-g68rx              1/1     Running   3                23m     192.168.56.22   worker-2   <none>           <none>
kube-flannel-ds-hkwqd              1/1     Running   1                23m     192.168.56.21   worker-1   <none>           <none>
kube-proxy-7tcjf                   1/1     Running   1                23m     192.168.56.21   worker-1   <none>           <none>
kube-proxy-j22bn                   1/1     Running   1                25m     192.168.56.12   master-2   <none>           <none>
kube-proxy-wq9hg                   1/1     Running   1                23m     192.168.56.22   worker-2   <none>           <none>
kube-proxy-xsrqt                   1/1     Running   3                30m     192.168.56.11   master-1   <none>           <none>
kube-scheduler-master-1            1/1     Running   4                30m     192.168.56.11   master-1   <none>           <none>
kube-scheduler-master-2            1/1     Running   1                25m     192.168.56.12   master-2   <none>           <none>
```

Now we can see the etcd isn’t present in the cluster, because it’s external.

Step 5: Deploy application

- Access to master node:

```sh
sshpass -p vagrant ssh vagrant@192.168.56.11
```

- Create file `demo-nginx.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 4
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector: 
    app: nginx
  type: NodePort  
  ports:
    - port: 80
      targetPort: 80
      nodePort: 32000
```

- Create pods:

```sh
kubectl apply -f demo-nginx.yaml
```

- Get pods status:

```sh
kubectl get pods -o wide
```

```
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
nginx-deployment-74d589986c-96lkf   1/1     Running   0          40s   10.244.3.3   worker-2   <none>           <none>
nginx-deployment-74d589986c-f8cgh   1/1     Running   0          40s   10.244.2.2   worker-1   <none>           <none>
nginx-deployment-74d589986c-pph6t   1/1     Running   0          40s   10.244.3.2   worker-2   <none>           <none>
nginx-deployment-74d589986c-vzn68   1/1     Running   0          40s   10.244.2.3   worker-1   <none>           <none>
```

- Access application from external service: `curl <workerIP>:<nodePort>`

```sh
curl http://192.168.56.21:32000
curl http://192.168.56.22:32000
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Certificate Rotation

## Auto rotation of Kubernetes certificates by DaemonSet

1. Run minikube, other-wise you can run a kubernetes cluster:

```sh
minikube start
```

1. Validate the cluster:

```sh
kubectl get pods -A
```

```
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE
kube-system   coredns-6d4b75cb6d-n96gg           1/1     Running   0             18m
kube-system   etcd-minikube                      1/1     Running   0             18m
kube-system   kube-apiserver-minikube            1/1     Running   0             18m
kube-system   kube-controller-manager-minikube   1/1     Running   0             18m
kube-system   kube-proxy-r7tkh                   1/1     Running   0             18m
kube-system   kube-scheduler-minikube            1/1     Running   0             18m
kube-system   storage-provisioner                1/1     Running   1 (17m ago)   18m
```

1. Create a symlink in the minikube container:

Firstly, ssh to the minikube container:

```sh
minikube ssh
```

```
docker@minikube:~$
```

In the minikube container, we create a symlink. If you run a kubernetes cluster, you can skip this.

```sh
sudo ln -s /var/lib/minikube/binaries/v1.24.3/kube* /usr/bin/
sudo mkdir -p /etc/kubernetes/pki
sudo ln -s /var/lib/minikube/certs/* /etc/kubernetes/pki/
```

Exit the container:

```sh
<Ctrl> + D
```

1. Check certificate expiration:

```sh
docker exec -it minikube kubeadm certs check-expiration
```

```
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Jan 13, 2024 17:04 UTC   364d            ca                      no      
apiserver                  Jan 12, 2026 17:04 UTC   2y              ca                      no      
apiserver-etcd-client      Jan 13, 2024 17:04 UTC   364d            etcd-ca                 no      
apiserver-kubelet-client   Jan 13, 2024 17:04 UTC   364d            ca                      no      
controller-manager.conf    Jan 13, 2024 17:04 UTC   364d            ca                      no      
etcd-healthcheck-client    Jan 13, 2024 17:04 UTC   364d            etcd-ca                 no      
etcd-peer                  Jan 13, 2024 17:04 UTC   364d            etcd-ca                 no      
etcd-server                Jan 13, 2024 17:04 UTC   364d            etcd-ca                 no      
front-proxy-client         Jan 13, 2024 17:04 UTC   364d            front-proxy-ca          no      
scheduler.conf             Jan 13, 2024 17:04 UTC   364d            ca                      no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Jan 10, 2033 17:04 UTC   9y              no      
etcd-ca                 Jan 10, 2033 17:04 UTC   9y              no      
front-proxy-ca          Jan 10, 2033 17:04 UTC   9y              no
```

We can see the expire date is `Jan 13, 2024 17:04 UTC`.

1. Create the DaemonSet:

Now we will setup a DaemonSet to rotate the kubernetes certificates by applying the manifests:

```sh
kubectl apply -f manifests/
```

```
daemonset.apps/kucero created
clusterrole.rbac.authorization.k8s.io/kucero created
clusterrolebinding.rbac.authorization.k8s.io/kucero created
role.rbac.authorization.k8s.io/kucero created
rolebinding.rbac.authorization.k8s.io/kucero created
serviceaccount/kucero created
```

Validate the DaemonSet:

```sh
kubectl get ds -n kube-system kucero
```

```
NAME     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kucero   1         1         1       1            1           <none>          3m56s
```

1. Tracking:

Every 5 minutes:

```sh
docker exec -it minikube kubeadm certs check-expiration
```

```
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Jan 13, 2024 17:45 UTC   364d            ca                      no      
apiserver                  Jan 12, 2026 17:04 UTC   2y              ca                      no      
apiserver-etcd-client      Jan 13, 2024 17:45 UTC   364d            etcd-ca                 no      
apiserver-kubelet-client   Jan 13, 2024 17:45 UTC   364d            ca                      no      
controller-manager.conf    Jan 13, 2024 17:46 UTC   364d            ca                      no      
etcd-healthcheck-client    Jan 13, 2024 17:45 UTC   364d            etcd-ca                 no      
etcd-peer                  Jan 13, 2024 17:46 UTC   364d            etcd-ca                 no      
etcd-server                Jan 13, 2024 17:45 UTC   364d            etcd-ca                 no      
front-proxy-client         Jan 13, 2024 17:46 UTC   364d            front-proxy-ca          no      
scheduler.conf             Jan 13, 2024 17:45 UTC   364d            ca                      no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Jan 10, 2033 17:04 UTC   9y              no      
etcd-ca                 Jan 10, 2033 17:04 UTC   9y              no      
front-proxy-ca          Jan 10, 2033 17:04 UTC   9y              no
```

The expire date now is `Jan 13, 2024 17:45 UTC`.

Every 5 minutes:

```sh
docker exec -it minikube kubeadm certs check-expiration
```

```
[check-expiration] Reading configuration from the cluster...
[check-expiration] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'

CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Jan 13, 2024 17:52 UTC   364d            ca                      no      
apiserver                  Jan 12, 2026 17:04 UTC   2y              ca                      no      
apiserver-etcd-client      Jan 13, 2024 17:52 UTC   364d            etcd-ca                 no      
apiserver-kubelet-client   Jan 13, 2024 17:52 UTC   364d            ca                      no      
controller-manager.conf    Jan 13, 2024 17:52 UTC   364d            ca                      no      
etcd-healthcheck-client    Jan 13, 2024 17:52 UTC   364d            etcd-ca                 no      
etcd-peer                  Jan 13, 2024 17:52 UTC   364d            etcd-ca                 no      
etcd-server                Jan 13, 2024 17:52 UTC   364d            etcd-ca                 no      
front-proxy-client         Jan 13, 2024 17:52 UTC   364d            front-proxy-ca          no      
scheduler.conf             Jan 13, 2024 17:52 UTC   364d            ca                      no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Jan 10, 2033 17:04 UTC   9y              no      
etcd-ca                 Jan 10, 2033 17:04 UTC   9y              no      
front-proxy-ca          Jan 10, 2033 17:04 UTC   9y              no
```

The expire date now is `Jan 13, 2024 17:52 UTC`.

That’s working!

### Contexts and Multiple Clusters

#### Configure Access to multiple clusters

Suppose we have a minikube cluster on local machine, we can get the content of config file from: `$HOME/.kube/config`

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/ph3/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Sat, 23 Apr 2022 22:15:17 EDT
        provider: minikube.sigs.k8s.io
        version: v1.24.0
      name: cluster_info
    server: https://192.168.49.2:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Sat, 23 Apr 2022 22:15:17 EDT
        provider: minikube.sigs.k8s.io
        version: v1.24.0
      name: context_info
    namespace: default
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /home/ph3/.minikube/profiles/minikube/client.crt
    client-key: /home/ph3/.minikube/profiles/minikube/client.key
```

Show all contexts and current context:

```
$ kubectl config get-contexts                                                                
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         minikube   minikube   minikube   default

$ kubectl config current-context 
minikube
```

Suppose we have another kubernetes cluster with information about `certificate-authority-data` and `server` also `client-certificate-data` and `client-key-data`. If we want to access to this cluster from local machine, we need to change config file, look like following that:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/ph3/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Sat, 23 Apr 2022 22:15:17 EDT
        provider: minikube.sigs.k8s.io
        version: v1.24.0
      name: cluster_info
    server: https://192.168.49.2:8443
  name: minikube
- cluster:
    certificate-authority-data: xxxxxx
    server: xxxxxx
  name: kubernetes
contexts:
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Sat, 23 Apr 2022 22:15:17 EDT
        provider: minikube.sigs.k8s.io
        version: v1.24.0
      name: context_info
    namespace: default
    user: minikube
  name: minikube
- context:
    cluster: kubernetes
    namespace: default
    user: dev-admin
  name: dev-admin@kubernetes
current-context: minikube
kind: Config
preferences: {}
users:
- name: minikube
  user:
    client-certificate: /home/ph3/.minikube/profiles/minikube/client.crt
    client-key: /home/ph3/.minikube/profiles/minikube/client.key
- name: dev-admin
  user:
    client-certificate-data: xxxxxx
    client-key-data: xxxxxx
```

Filling data from `certificate-authority-data`, `server`, `client-certificate-data` and `client-key-data` into xxxxxx.

Now we listing contexts, then switching between contexts:

```
$ kubectl config get-contexts
CURRENT   NAME                   CLUSTER      AUTHINFO    NAMESPACE
          dev-admin@kubernetes   kubernetes   dev-admin   default
*         minikube               minikube     minikube    default

$ kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
minikube   Ready    control-plane,master   87d   v1.22.3

$ kubectl config use-context dev-admin@kubernetes
Switched to context "dev-admin@kubernetes".

$ kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
master-1   Ready    control-plane,master   19d   v1.23.0
master-2   Ready    control-plane,master   19d   v1.23.0
worker-1   Ready    <none>                 19d   v1.23.0
worker-2   Ready    <none>                 19d   v1.23.0
```

You can get full source code here: [make-basic-cluster](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/make-basic-cluster), [make-stacked-etcd-cluster](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/make-stacked-etcd-cluster), [make-external-etcd-cluster](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/make-external-etcd-cluster), [certificate-rotation](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/certificate-rotation), [contexts-multiple-clusters](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/contexts-multiple-clusters).
