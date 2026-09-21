---
title: "Kubernetes Configuration and Storage"
date: 2023-09-23T16:14:26+07:00
tags: ["Kubernetes", "DevOps"]
author: ["hoangph3"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Manage application configuration with ConfigMaps and Secrets, and persist data using Kubernetes volumes."
summary: "Manage application configuration with ConfigMaps and Secrets, and persist data using Kubernetes volumes."
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

## ConfigMaps and Secrets

### Configure all key-value pairs as environment variables

```sh
kubectl apply -f config-env-vars-envFrom.yaml
```

```
configmap/myapp-config created
secret/myapp-secret created
deployment.apps/my-app created
```

```sh
kubectl get pods
```

```
NAME                      READY   STATUS      RESTARTS      AGE
my-app-6594549577-7s7ks   0/1     Completed   3 (32s ago)   60s
```

```sh
kubectl logs -f my-app-6594549577-7s7ks
```

```
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT=443
HOSTNAME=my-app-6594549577-7s7ks
SHLVL=1
username=admin
HOME=/root
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
password=admin
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
db_host=mysql-service
```

### Configure defined environment variables in the `command` and `args` of a container using the $(VAR_NAME)

```sh
kubectl apply -f config-env-vars-valueFrom.yaml 
```

```
configmap/myapp-config created
secret/myapp-secret created
deployment.apps/my-app created
```

```sh
kubectl get pods
```

```
NAME                      READY   STATUS      RESTARTS   AGE
my-app-6df7cd5d47-dhd2c   0/1     Completed   0          6s
```

```sh
kubectl logs -f my-app-6df7cd5d47-dhd2c
```

```
admin admin mysql-service
```

### Configure as a Volume

```sh
kubectl apply -f config-volumes.yaml
```

```
configmap/mysql-config created
secret/mysql-secret created
deployment.apps/my-db created
```

```sh
kubectl get pods
```

```
NAME                     READY   STATUS              RESTARTS   AGE
my-db-569cdd7c6c-mrshr   0/1     ContainerCreating   0          3s
```

```sh
kubectl logs -f my-db-569cdd7c6c-mrshr
```

```
/mysql/db-config/mysql.conf
[mysqld]
port=3306
socket=/tmp/mysql.sock
key_buffer_size=16M
max_allowed_packet=128M

/mysql/db-config/secure-flag
ThiS_Is_FLagggggggggg_4U@@

/mysql/db-config/test.conf
ThiS_iS_0nLy_f0R_T3st!^^

/mysql/db-secret/secret.file
Sup3r_s3cure_F1agggggggggg!^^
```

Because we omit the items array entirely, every key in the ConfigMap and Secret becomes a file with the same name as the key. So we get 4 files, contain 3 files from ConfigMap and 1 file from Secret.

### Configure as a Volume with items

```sh
kubectl apply -f config-volumes-with-items.yaml
kubectl get pods
kubectl logs -f my-db-5f9585df5f-8fzlc
```

```
configmap/mysql-config created
secret/mysql-secret created
deployment.apps/my-db created

NAME                     READY   STATUS      RESTARTS   AGE
my-db-5f9585df5f-8fzlc   0/1     Completed   0          8s

/mysql/db-config/flag.txt
ThiS_Is_FLagggggggggg_4U@@

/mysql/db-config/test.conf
ThiS_iS_0nLy_f0R_T3st!^^

/mysql/db-secret/flag.txt
Sup3r_s3cure_F1agggggggggg!^^
```

We defined 2 arrays of keys from the ConfigMap (not contain mysql.conf) and 1 array from Secret to create as files, the filename was changed from `key` to `path` (default is `key`).

### Configure Redis

```sh
kubectl apply -f config-redis.yaml
```

```
configmap/example-redis-config created
deployment.apps/my-redis created
service/my-redis-service created
```

```sh
kubectl get svc -o wide
kubectl get pods
```

```
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE   SELECTOR
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP          65d   <none>
my-redis-service   NodePort    10.105.172.53   <none>        6379:30100/TCP   35s   app=my-redis

NAME                        READY   STATUS    RESTARTS   AGE
my-redis-6496f6bbf8-nsgjk   1/1     Running   0          40s
```

Access redis server to get config and data:

```sh
kubectl exec -it my-redis-6496f6bbf8-nsgjk -- redis-cli
```

```
127.0.0.1:6379> CONFIG GET maxmemory
1) "maxmemory"
2) "2097152"

127.0.0.1:6379> CONFIG GET maxmemory-policy
1) "maxmemory-policy"
2) "allkeys-lru"

127.0.0.1:6379> keys *
(empty array)
```

Now we will create python script `test_redis.py` to communicate with redis server:

```python
# test_redis.py
import redis

r = redis.Redis(host="192.168.49.2", # host is url that kubernetes control plane is running.
                port="30100",
                db=0)
r.rpush('foo', 'bar')
r.rpush('foo', 'bar2')
```

Note that host argument is the url that kubernetes control plane is running. In this case, we use minikube and the url can find by the command `minikube ip` or `kubectl cluster-info`:

```
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Access redis server to get data after run python script:

```sh
python3 test_redis.py
kubectl exec -it my-redis-6496f6bbf8-nsgjk -- redis-cli
```

```
127.0.0.1:6379> keys *
1) "foo"

127.0.0.1:6379> lrange foo 0 -1
1) "bar"
2) "bar2"
```

### Pass credentials for the Docker registry with Secret

First, creat a Secret holding the credentials for authenticating with a Docker registry:

```
kubectl create secret docker-registry mydockerhubsecret --docker-username=hoangph3 --docker-password=mypassword --docker-email=hoangph3@example.com
```

Let’s run pod with private image:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-pod
spec:
  imagePullSecrets:
  - name: mydockerhubsecret
  containers:
  - image: hoangph3/python-docker:v1.0
    name: myapp
```

```sh
kubectl apply -f secret-private-image.yaml
kubectl describe pod private-pod
```

```
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  63s               default-scheduler  Successfully assigned default/private-pod to minikube
  Normal   Pulling    62s               kubelet            Pulling image "hoangph3/python-docker:v1.0"
  Normal   Pulled     46s               kubelet            Successfully pulled image "hoangph3/python-docker:v1.0" in 16.430969904s
  Normal   Created    2s (x4 over 45s)  kubelet            Created container myapp
  Normal   Started    2s (x4 over 45s)  kubelet            Started container myapp
```

## Data Persistence

### Volume

### Share data between containers with `emptyDir`

Step 1: Create deployment with emptyDir volume:

```sh
kubectl apply -f emptydir-volume.yaml
```

Step 2: Tracking logs in pods:

```sh
kubectl get pods
kubectl logs -f my-app-68f4bfd84c-79wvl log-sidecar
```

```
NAME                      READY   STATUS    RESTARTS   AGE
my-app-68f4bfd84c-79wvl   2/2     Running   0          9s

Thu Apr  7 14:44:10 UTC 2022 INFO some app data
Thu Apr  7 14:44:15 UTC 2022 INFO some app data
Thu Apr  7 14:44:20 UTC 2022 INFO some app data
Thu Apr  7 14:44:25 UTC 2022 INFO some app data
Thu Apr  7 14:44:30 UTC 2022 INFO some app data
Thu Apr  7 14:44:35 UTC 2022 INFO some app data
Thu Apr  7 14:44:40 UTC 2022 INFO some app data
```

### `PersistentVolumeClaims` and `PersistentVolumes`

The `PersistentVolumes` is resource that communicate with Storage, and the `PersistentVolumeClaims` request resource from `PersistentVolumes`. In production environment, the administrator will create the cluster, install plugin, … while the developers will write yaml file to deploy application. So, the `PersistentVolumes` will created by the administrator, the developers only need to create `PersistentVolumeClaims` to use.

Suppose you are administrator, you will create the `PersistentVolumes`:

```sh
kubectl apply -f pv.yaml
kubectl get pv
```

```
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
data-pv   10Gi       RWO            Retain           Available                                   59s
```

Note that the `PersistentVolumes` is not belong to any namespace, this is cluster resource, same as node. But the Pod, Deployment, … is the namespace resource.

Now, suppose you are developers, you need to create `PersistentVolumeClaims` to store persistent data. If exist any `PersistentVolumes`, the `PersistentVolumeClaims` you created will request storage from it.

```sh
kubectl apply -f pvc.yaml
kubectl get pvc
```

```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-data-pvc   Bound    pvc-e5d8e277-3831-4a1b-b9c4-351df960f58a   5Gi        RWO            standard       7s
```

The STATUS=Bound indicate that the `mysql-data-pvc` bounded `pvc-e5d8e277-3831-4a1b-b9c4-351df960f58a` volume, now let’s show the pv:

```sh
kubectl get pv
```

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                    STORAGECLASS   REASON   AGE
data-pv                                    10Gi       RWO            Retain           Available                                                    119s
pvc-e5d8e277-3831-4a1b-b9c4-351df960f58a   5Gi        RWO            Delete           Bound       default/mysql-data-pvc   standard                31s
```

The `default/mysql-data-pvc` pvc was claimed resource from `pvc-e5d8e277-3831-4a1b-b9c4-351df960f58a` pv.

You can get full source code here: [configmap-secret](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/configmap-secret), [data-persistence](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/data-persistence).
