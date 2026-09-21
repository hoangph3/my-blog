---
title: "Kubernetes Workloads and Scheduling"
date: 2023-09-23T16:14:26+07:00
tags: ["Kubernetes", "DevOps"]
author: ["hoangph3"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Control how Kubernetes schedules and manages workloads: Jobs, StatefulSets, taints/tolerations, affinity rules, health probes and resource limits."
summary: "Control how Kubernetes schedules and manages workloads: Jobs, StatefulSets, taints/tolerations, affinity rules, health probes and resource limits."
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

## Kubernetes Jobs

### Running a single completable task by Job

```yaml
# job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  template:
    metadata:
      labels:
        app: my-job
    spec:
      containers:
      - name: my-job
        image: busybox
        command: ['sh', '-c']
        args: 
        - echo "$(date) Job starting";
          sleep 30;
          echo "$(date) Finished succesfully";
      restartPolicy: OnFailure
```

The YAML defines a Job that will run the busybox image, which invokes a process that runs for exactly 30 seconds and then exits. In spec section, the `restartPolicy` is set `OnFailure`, because Job can’t use the default restart policy (which is `Always`). This setting is what prevents the container from being restarted when it finishes.

After we run command `kubectl apply -f job.yaml` to create job, we can list pods by `kubectl get pods`:

```
NAME              READY   STATUS    RESTARTS   AGE
my-job--1-4n4rk   1/1     Running   0          8s
```

After 30 seconds have passed, the job has completed and no restart:

```
NAME              READY   STATUS      RESTARTS   AGE
my-job--1-4n4rk   0/1     Completed   0          39s
```

We can get log of job:

```sh
kubectl logs -f my-job--1-4n4rk
```

```
Sat Apr  9 15:34:32 UTC 2022 Job starting
Sat Apr  9 15:35:02 UTC 2022 Finished succesfully
```

To running job pods sequentially or parallel, we can configure the `.spec.completions` and the `.spec.parallelism`:

```yaml
# batch-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  completions: 5  # this job must ensure 5 pods complete successfully
  parallelism: 2  # up to 2 pods can run in parallel
  template:
    metadata:
      labels:
        app: my-job
    spec:
      containers:
      - name: my-job
        image: busybox
        command: ['sh', '-c']
        args: 
        - echo "$(date) Job starting";
          sleep 30;
          echo "$(date) Finished succesfully";
      restartPolicy: OnFailure
```

With `completions: 5` indicate that this job must ensure 5 pods complete successfully, and up to 2 pods can run in parallel with `parallelism: 2`.

```sh
kubectl get pods
kubectl get jobs
```

```
NAME              READY   STATUS    RESTARTS   AGE
my-job--1-grq6l   1/1     Running   0          9s
my-job--1-w85p2   1/1     Running   0          9s

NAME     COMPLETIONS   DURATION   AGE
my-job   0/5           17s        17s
```

When the job completed:

```sh
kubectl get pods
kubectl get jobs
```

```
NAME              READY   STATUS      RESTARTS   AGE
my-job--1-cv5js   0/1     Completed   0          77s
my-job--1-grq6l   0/1     Completed   0          112s
my-job--1-khnbl   0/1     Completed   0          73s
my-job--1-w85p2   0/1     Completed   0          112s
my-job--1-x95mk   0/1     Completed   0          41s

NAME     COMPLETIONS   DURATION   AGE
my-job   5/5           107s       115s
```

### Scheduling Jobs to run periodically

Let’s create a CronJob:

```yaml
# cron-job.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-every-minute
spec:
  schedule: "* * * * *"
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: cronjob-every-minute
        spec:
          containers:
          - name: cronjob-every-minute
            image: busybox
            command: ['sh', '-c']
            args: 
            - echo "$(date) Job starting";
              sleep 30;
              echo "$(date) Finished succesfully";
          restartPolicy: OnFailure
```

```sh
kubectl apply -f cron-job.yaml
kubectl get cronjobs.batch
```

```
NAME                   SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob-every-minute   * * * * *   False     1        8s              44s
```

After 3 minutes, let’s list of pods:

```sh
kubectl get pods
```

```
NAME                                     READY   STATUS      RESTARTS   AGE
cronjob-every-minute-27492030--1-qdtgl   0/1     Completed   0          2m38s
cronjob-every-minute-27492031--1-wl9jr   0/1     Completed   0          98s
cronjob-every-minute-27492032--1-rb6p6   0/1     Completed   0          38s
```

Print log of pods, we can see the job is scheduled every minute:

```sh
kubectl logs -f cronjob-every-minute-27492030--1-qdtgl
kubectl logs -f cronjob-every-minute-27492031--1-wl9jr
kubectl logs -f cronjob-every-minute-27492032--1-rb6p6
```

```
Sat Apr  9 16:30:04 UTC 2022 Job starting
Sat Apr  9 16:30:34 UTC 2022 Finished succesfully

Sat Apr  9 16:31:04 UTC 2022 Job starting
Sat Apr  9 16:31:34 UTC 2022 Finished succesfully

Sat Apr  9 16:32:04 UTC 2022 Job starting
Sat Apr  9 16:32:34 UTC 2022 Finished succesfully
```

The `.spec.successfulJobsHistoryLimit` and `.spec.failedJobsHistoryLimit` fields are optional. These fields specify how many completed and failed jobs should be kept. By default, they are set to 3 and 1 respectively.

Access to [https://crontab.guru/](https://crontab.guru/) to get more info about cronjob schedule expressions.

## StatefulSets

### StatefulSet Application

The StatefulSet look like the ReplicaSet, but it’s creates both Pods and PersistentVolumeClaims. And a StatefulSet maintains a sticky identity for each of their Pods. StatefulSet Pods have a unique identity that is comprised of an ordinal, a stable network identity, and stable storage.

For a StatefulSet with N replicas, each Pod in the StatefulSet will be assigned an integer ordinal, from 0 up through N-1, that is unique over the Set, each Pod is attached to a PersistentVolumeClaim.

#### Create the PersistentVolume

Because the PersistentVolumeClaim will request resource from PersistentVolume, so we need create PersistentVolume first. Note that we must create more if we plan on scaling the StatefulSet up more than that.

In this tutorial, we will need 3 PersistentVolumes, because we will be scaling the StatefulSet up to 3 replicas.

```sh
kubectl apply -f persistent-volumes-hostpath.yaml
kubectl get pv
```

```
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-a   1Mi        RWO            Retain           Available                                   7s
pv-b   1Mi        RWO            Retain           Available                                   7s
pv-c   1Mi        RWO            Retain           Available                                   7s
```

Create StatefulSet with Headless Service:

```sh
kubectl apply -f kubia-statefulset.yaml
kubectl get pods
```

```
NAME      READY   STATUS    RESTARTS   AGE
kubia-0   1/1     Running   0          104s
kubia-1   1/1     Running   0          56s
```

Forward port to test connection from localhost:

```sh
kubectl port-forward kubia-0 8080:8080
curl localhost:8080
```

```
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080

You've hit kubia-0
Data stored on this pod: No data posted yet
```

Send HTTP POST request to the application:

```sh
curl -X POST -d "Hello kubia-0" localhost:8080
```

```
Data stored on pod kubia-0
```

Send HTTP GET request to the application:

```sh
curl localhost:8080
```

```
You've hit kubia-0
Data stored on this pod: Hello kubia-0
```

Now let’s interact with another pod:

```sh
kubectl port-forward kubia-0 8081:8080
curl localhost:8081
```

```
You've hit kubia-1
Data stored on this pod: No data posted yet
```

As expected, each node has its own state. But is that state persisted?

We are going to delete the kubia-0 pod and wait for it to be rescheduled. Then we will see if it’s still serving the same data as before.

```sh
kubectl delete pods kubia-0
kubectl get pods
```

```
NAME      READY   STATUS        RESTARTS   AGE
kubia-0   1/1     Terminating   0          20m
kubia-1   1/1     Running       0          20m
```

The pod is rescheduled:

```sh
kubectl get pods
```

```
NAME      READY   STATUS              RESTARTS   AGE
kubia-0   0/1     ContainerCreating   0          5s
kubia-1   1/1     Running             0          21m

NAME      READY   STATUS    RESTARTS   AGE
kubia-0   1/1     Running   0          8s
kubia-1   1/1     Running   0          21m
```

Now we will use API server to communicate the kubia-0, it’s another option like forward port:

First, run proxy:

```sh
kubectl proxy
```

Send HTTP GET request to the URL with template: `<apiServerHost>:<port>/api/v1/namespaces/default/pods/kubia-0/proxy/<path>`

```
curl 127.0.0.1:8001/api/v1/namespaces/default/pods/kubia-0/proxy/
```

```
You've hit kubia-0
Data stored on this pod: Hello kubia-0
```

It’s data persistence!

## Taints and Tolerations

### Configure Taints and Tolerations

We use a taint to prevent pods from being scheduled to the master node or worker node, unless those pods tolerate this taint. The pods that tolerate it are usually system pods.

Suppose you have a cluster with 2 master nodes and 2 worker nodes:

```
vagrant@master-1:~$ kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
master-1   Ready    control-plane,master   16d   v1.23.0
master-2   Ready    control-plane,master   16d   v1.23.0
worker-1   Ready    <none>                 16d   v1.23.0
worker-2   Ready    <none>                 16d   v1.23.0
```

In kubernetes cluster, default you can only deploy your pods to the worker nodes (not master nodes), unless the kube-system pods. Because the master nodes have taints, and the kube-system pods tolerates the master nodes’s taints.

Let’s get taints from master nodes:

```
vagrant@master-1:~$ kubectl describe node master-1
Name:               master-1
Roles:              control-plane,master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=master-1
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
                    node-role.kubernetes.io/master=
                    node.kubernetes.io/exclude-from-external-load-balancers=
Annotations:        flannel.alpha.coreos.com/backend-data: {"VNI":1,"VtepMAC":"42:7b:34:b8:33:95"}
                    flannel.alpha.coreos.com/backend-type: vxlan
                    flannel.alpha.coreos.com/kube-subnet-manager: true
                    flannel.alpha.coreos.com/public-ip: 10.0.2.15
                    kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Tue, 05 Apr 2022 07:00:15 +0000
Taints:             node-role.kubernetes.io/master:NoSchedule
...
```

The format of Taints is `<key>=<value>:<effect>`. Only the kube-system pods with `Tolerations: node-role.kubernetes.io/master=:NoSchedule` can be scheduled on the master nodes.

Let’s describe the kube-system pods:

```
vagrant@master-1:~$ kubectl get po -n kube-system
NAME                               READY   STATUS    RESTARTS         AGE
coredns-64897985d-fppdp            1/1     Running   5 (9m13s ago)    16d
coredns-64897985d-t4mxq            1/1     Running   5 (9m13s ago)    16d
kube-apiserver-master-1            1/1     Running   6 (9m13s ago)    16d
kube-apiserver-master-2            1/1     Running   2 (8m23s ago)    16d
kube-controller-manager-master-1   1/1     Running   8 (5m7s ago)     16d
kube-controller-manager-master-2   1/1     Running   4 (8m23s ago)    16d
kube-flannel-ds-4qtvn              1/1     Running   4 (8m23s ago)    16d
kube-flannel-ds-fxptk              1/1     Running   14 (9m13s ago)   16d
kube-flannel-ds-g68rx              1/1     Running   5 (6m36s ago)    16d
kube-flannel-ds-hkwqd              1/1     Running   4 (4m59s ago)    16d
kube-proxy-7tcjf                   1/1     Running   3 (7m22s ago)    16d
kube-proxy-j22bn                   1/1     Running   3 (8m23s ago)    16d
kube-proxy-wq9hg                   1/1     Running   3 (6m36s ago)    16d
kube-proxy-xsrqt                   1/1     Running   5 (9m13s ago)    16d
kube-scheduler-master-1            1/1     Running   8 (5m10s ago)    16d
kube-scheduler-master-2            1/1     Running   3 (8m23s ago)    16d

vagrant@master-1:~$ kubectl describe pod -n kube-system | grep Tolerations
Tolerations:                 CriticalAddonsOnly op=Exists
Tolerations:                 CriticalAddonsOnly op=Exists
Tolerations:       :NoExecute op=Exists
Tolerations:       :NoExecute op=Exists
Tolerations:       :NoExecute op=Exists
Tolerations:       :NoExecute op=Exists
Tolerations:                 :NoSchedule op=Exists
Tolerations:                 :NoSchedule op=Exists
Tolerations:                 :NoSchedule op=Exists
Tolerations:                 :NoSchedule op=Exists
Tolerations:                 op=Exists
Tolerations:                 op=Exists
Tolerations:                 op=Exists
Tolerations:                 op=Exists
Tolerations:       :NoExecute op=Exists
Tolerations:       :NoExecute op=Exists
```

Each taint has an effect associated with it. Three possible effects exist:

- `NoSchedule`: pods won’t be scheduled to the node if they don’t tolerate the taint.
- `PreferNoSchedule` is a soft version of `NoSchedule`, meaning the scheduler will try to avoid scheduling the pod to the node, but will schedule it to the node if it can’t schedule it somewhere else.
- `NoExecute`, unlike `NoSchedule` and `PreferNoSchedule` that only affect scheduling, also affects pods already running on the node. If you add a `NoExecute` taint to a node, pods that are already running on that node and don’t tolerate the `NoExecute` taint will be evicted from the node.

#### Add Taints to Nodes

Imagine having a single Kubernetes cluster where you run both production and development workloads. It’s of the utmost importance that development pods never run on the production nodes. This can be achieved by adding a taint to your production nodes.

```
vagrant@master-1:~$ kubectl get nodes
NAME       STATUS   ROLES                  AGE   VERSION
master-1   Ready    control-plane,master   16d   v1.23.0
master-2   Ready    control-plane,master   16d   v1.23.0
worker-1   Ready    <none>                 16d   v1.23.0
worker-2   Ready    <none>                 16d   v1.23.0

vagrant@master-1:~$ kubectl taint node worker-1 node-type=production:NoSchedule
node/worker-1 tainted
```

This adds a taint with key node-type, value production and the `NoSchedule` effect. If you now deploy multiple replicas of a regular pod, you’ll see none of them are scheduled to the node you tainted, as shown in the following listing.

```
vagrant@master-1:~$ kubectl create deploy test --image busybox --replicas 5 -- sleep 99999
deployment.apps/test created

vagrant@master-1:~$ kubectl get pods -o wide
NAME                    READY   STATUS    RESTARTS   AGE    IP            NODE       NOMINATED NODE   READINESS GATES
test-5c4f786f47-59hf5   1/1     Running   0          102s   10.244.3.8    worker-2   <none>           <none>
test-5c4f786f47-h6ttk   1/1     Running   0          102s   10.244.3.9    worker-2   <none>           <none>
test-5c4f786f47-jr92r   1/1     Running   0          102s   10.244.3.12   worker-2   <none>           <none>
test-5c4f786f47-qvt4r   1/1     Running   0          102s   10.244.3.10   worker-2   <none>           <none>
test-5c4f786f47-r2r7z   1/1     Running   0          102s   10.244.3.11   worker-2   <none>           <none>
```

To deploy production pods to the production nodes, they need to tolerate the taint you added to the nodes, look like `pod-with-toleration.yaml` file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-with-toleration
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
      tolerations:
      - key: node-type
        operator: Equal
        value: production
        effect: NoSchedule
```

```
vagrant@master-1:~$ kubectl apply -f pod-with-tolerations.yaml 
deployment.apps/pod-with-toleration created

vagrant@master-1:~$ kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE    IP            NODE       NOMINATED NODE   READINESS GATES
pod-with-toleration-6c884d5b5b-6bjs4   1/1     Running   0          37s    10.244.3.13   worker-2   <none>           <none>
pod-with-toleration-6c884d5b5b-8zx8q   1/1     Running   0          37s    10.244.2.9    worker-1   <none>           <none>
pod-with-toleration-6c884d5b5b-c6jkl   1/1     Running   0          37s    10.244.2.10   worker-1   <none>           <none>
pod-with-toleration-6c884d5b5b-gb6rn   1/1     Running   0          37s    10.244.2.8    worker-1   <none>           <none>
pod-with-toleration-6c884d5b5b-hhcvf   1/1     Running   0          37s    10.244.3.14   worker-2   <none>           <none>
```

You can also use a toleration to specify how long Kubernetes should wait before rescheduling a pod to another node if the node that the pod is running on becomes unready or unreachable. Let’s see the tolerations of one of your pods:

```
vagrant@master-1:~$ kubectl get pod pod-with-toleration-6c884d5b5b-hhcvf -o yaml
apiVersion: v1
kind: Pod
...
spec:
  ...
  tolerations:
  - effect: NoSchedule
    key: node-type
    operator: Equal
    value: production
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
...
```

These tolerations say that this pod tolerates a node being `notReady` or `unreachable` for `300` seconds. The Kubernetes Control Plane, when it detects that a node is no longer ready or no longer reachable, will wait for `300` seconds before it deletes the pod and reschedules it to another node.

Finally, remove taints from node:

```
vagrant@master-1:~$ kubectl taint node worker-1 node-type=production:NoSchedule-
node/worker-1 untainted
```

## Node and Pod Affinity

### Node Affinity

The `nodeAffinity` allows you to tell Kubernetes to schedule pods only to specific subsets of nodes. Each pod can define its own `nodeAffinity` rules. These allow you to specify either hard requirements or preferences. By specifying a preference, you tell Kubernetes which nodes you prefer for a specific pod, and Kubernetes will try to schedule the pod to one of those nodes. If that’s not possible, it will choose one of the other nodes (the `nodeSelector` is not).

List the nodes in your cluster, along with their labels:

```
vagrant@master-1:~$ kubectl get nodes --show-labels
NAME       STATUS   ROLES                  AGE   VERSION   LABELS
master-1   Ready    control-plane,master   17d   v1.23.0   ...,kubernetes.io/hostname=master-1,...
master-2   Ready    control-plane,master   17d   v1.23.0   ...,kubernetes.io/hostname=master-2,...
worker-1   Ready    <none>                 17d   v1.23.0   ...,kubernetes.io/hostname=worker-1,kubernetes.io/os=linux
worker-2   Ready    <none>                 17d   v1.23.0   ...,kubernetes.io/hostname=worker-2,kubernetes.io/os=linux
```

Chose one of your nodes, and add a label to it:

```
vagrant@master-1:~$ kubectl label nodes worker-1 device=gpu
node/worker-1 labeled

vagrant@master-1:~$ kubectl label nodes worker-2 device=cpu
node/worker-2 labeled

vagrant@master-1:~$ kubectl get nodes --show-labels
NAME       STATUS   ROLES                  AGE   VERSION   LABELS
master-1   Ready    control-plane,master   17d   v1.23.0   ...,kubernetes.io/hostname=master-1,...
master-2   Ready    control-plane,master   17d   v1.23.0   ...,kubernetes.io/hostname=master-2,...
worker-1   Ready    <none>                 17d   v1.23.0   ...,device=gpu,kubernetes.io/hostname=worker-1,kubernetes.io/os=linux
worker-2   Ready    <none>                 17d   v1.23.0   ...,device=cpu,kubernetes.io/hostname=worker-2,kubernetes.io/os=linux
```

Schedule a Pod using required node affinity with `pod-required-node-affinity.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-required-node-affinity
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: device
                operator: In
                values:
                - gpu
      containers:
      - name: nginx
        image: nginx
```

The `requiredDuringSchedulingIgnoredDuringExecution` means that the pod will get scheduled only on a node that has a `device=gpu` label.

```
vagrant@master-1:~$ kubectl apply -f pod-required-node-affinity.yaml
deployment.apps/pod-required-node-affinity created

vagrant@master-1:~$ kubectl get pods -o wide
NAME                                          READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
pod-required-node-affinity-6fbcb8c97c-cvrwv   1/1     Running   0          2m24s   10.244.2.14   worker-1   <none>           <none>
pod-required-node-affinity-6fbcb8c97c-hwhg5   1/1     Running   0          2m24s   10.244.2.17   worker-1   <none>           <none>
pod-required-node-affinity-6fbcb8c97c-kcrc6   1/1     Running   0          2m24s   10.244.2.16   worker-1   <none>           <none>
pod-required-node-affinity-6fbcb8c97c-m82d4   1/1     Running   0          2m24s   10.244.2.18   worker-1   <none>           <none>
pod-required-node-affinity-6fbcb8c97c-qstx5   1/1     Running   0          2m24s   10.244.2.15   worker-1   <none>           <none>
```

Schedule a Pod using preferred node affinity with `pod-preferred-node-affinity.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-preferred-node-affinity
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 80
            preference:
              matchExpressions:
              - key: device 
                operator: In
                values:
                - gpu
          - weight: 20
            preference:
              matchExpressions:
              - key: device
                operator: In
                values:
                - cpu
      containers:
      - name: nginx
        image: nginx
```

The `preferredDuringSchedulingIgnoredDuringExecution` means the first preference rule (`device=gpu`) is important by setting its weight to 80, whereas the second one is much less important (weight is set to 20 with `device=cpu`).

```
vagrant@master-1:~$ kubectl apply -f pod-preferred-node-affinity.yaml 
deployment.apps/pod-preferred-node-affinity created

vagrant@master-1:~$ kubectl get pods -o wide
NAME                                           READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
pod-preferred-node-affinity-66f89c6b4d-59src   1/1     Running   0          36s   10.244.3.27   worker-2   <none>           <none>
pod-preferred-node-affinity-66f89c6b4d-8dlwd   1/1     Running   0          36s   10.244.2.20   worker-1   <none>           <none>
pod-preferred-node-affinity-66f89c6b4d-br4s4   1/1     Running   0          36s   10.244.2.21   worker-1   <none>           <none>
pod-preferred-node-affinity-66f89c6b4d-s5sr9   1/1     Running   0          36s   10.244.2.19   worker-1   <none>           <none>
pod-preferred-node-affinity-66f89c6b4d-zn4b2   1/1     Running   0          36s   10.244.2.22   worker-1   <none>           <none>
```

### Pod Affinity

Inter-pod affinity and anti-affinity allow you to configure that a set of workloads should be co-located in the same defined topology, eg., the same node.

Imagine having a web application and an in-memory cache like redis pod. Having those pods deployed near to each other reduces latency and improves the performance of the app. In Kubernetes, you could use inter-pod affinity and anti-affinity to co-locate the web servers with the cache as much as possible by `podAffinity`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  selector:
    matchLabels:
      app: store
  replicas: 2
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: redis-server
        image: redis:3.2-alpine
```

In the above example Deployment for the redis cache, the replicas get the label `app=store`. The `podAntiAffinity` rule tells the scheduler to avoid placing multiple replicas with the `app=store` label on a single node. This creates each cache in a separate node. In this case, we have 2 replicas corresponding to 2 worker nodes.

```
vagrant@master-1:~$ kubectl apply -f redis-pod-anti-affinity.yaml 
deployment.apps/redis-cache created

vagrant@master-1:~$ kubectl get pods -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
redis-cache-6b7b79d589-mfkdh   1/1     Running   0          75s   10.244.2.23   worker-1   <none>           <none>
redis-cache-6b7b79d589-vzmlf   1/1     Running   0          74s   10.244.3.28   worker-2   <none>           <none>
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-store
  replicas: 2
  template:
    metadata:
      labels:
        app: web-store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

The above Deployment for the web servers creates replicas with the label `app=web-store`. The `podAffinity` rule tells the scheduler to place each replica on a node that has a Pod with the label `app=store`. The `podAntiAffinity` rule tells the scheduler to avoid placing multiple `app=web-store` servers on a single node.

```
vagrant@master-1:~$ kubectl apply -f webapp-pod-affinity.yaml 
deployment.apps/web-server created

vagrant@master-1:~$ kubectl get pods -o wide
NAME                           READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
redis-cache-6b7b79d589-mfkdh   1/1     Running   0          6m21s   10.244.2.23   worker-1   <none>           <none>
redis-cache-6b7b79d589-vzmlf   1/1     Running   0          6m20s   10.244.3.28   worker-2   <none>           <none>
web-server-f6798875f-b5nn2     1/1     Running   0          17s     10.244.2.24   worker-1   <none>           <none>
web-server-f6798875f-s625n     1/1     Running   0          16s     10.244.3.29   worker-2   <none>           <none>
```

Suppose we need scaling the replicas up to `3`, we will get the `Pending` status:

```
vagrant@master-1:~$ kubectl get pods -o wide
NAME                           READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
redis-cache-6b7b79d589-gvcdh   1/1     Running   0          2m54s   10.244.3.32   worker-2   <none>           <none>
redis-cache-6b7b79d589-hl44v   0/1     Pending   0          2m24s   <none>        <none>     <none>           <none>
redis-cache-6b7b79d589-pfxlc   1/1     Running   0          2m54s   10.244.2.27   worker-1   <none>           <none>
web-server-f6798875f-2j65x     1/1     Running   0          2m45s   10.244.3.33   worker-2   <none>           <none>
web-server-f6798875f-dp6pj     0/1     Pending   0          2m14s   <none>        <none>     <none>           <none>
web-server-f6798875f-k5595     1/1     Running   0          2m45s   10.244.2.28   worker-1   <none>           <none>
```

Let’s describe the `Pending` pod:

```
vagrant@master-1:~$ kubectl describe pod web-server-f6798875f-dp6pj
...
Events:
  Type     Reason            Age                    From               Message
  ----     ------            ----                   ----               -------
  Warning  FailedScheduling  8m51s                  default-scheduler  0/4 nodes are available: 2 node(s) didn't match pod anti-affinity rules, 2 node(s) had taint {node-role.kubernetes.io/master: }, that the pod didn't tolerate.
...
```

My cluster have 4 nodes with 2 master nodes and 2 worker nodes. With `podAntiAffinity`, the scheduler will create each pod in a separate node, we have `3` pods with `replicas: 3` (2 pods on 2 worker nodes, 1 pod on 1 master node). Because the master node have taint, the pod can’t be scheduled on it.

### Node Name

`nodeName` is a more direct form of node selection than `nodeAffinity` or `nodeSelector`. `nodeName` is a field in the Pod spec. If the `nodeName` field is not empty, the scheduler ignores the Pod and the kubelet on the named node tries to place the Pod on that node.

The `nodeName` have some limitations such as: the Pod will not run if the named node does not exist, the Pod will fail if the named node does not have the resources to accommodate the Pod, and node names in cloud environments are not always stable.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: worker-2
```

```
vagrant@master-1:~$ kubectl apply -f pod-with-node-name.yaml 
pod/nginx created

vagrant@master-1:~$ kubectl get pods -o wide
NAME    READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          14s   10.244.3.42   worker-2   <none>           <none>
```

## Pod Health Probes

### Configure Liveness, Readiness and Startup Probes

### Configure `livenessProbe` by execute command

```yaml
# exec-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec: # the kubelet executes the command to perform a probe
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5 # perform a liveness probe every 5 seconds
      periodSeconds: 5 # wait 5 seconds before performing the first probe
```

In `livenessProbe` section, the kubelet executes the command `cat /tmp/healthy` to perform a probe.

`initialDelaySeconds`: perform a liveness probe every 5 seconds.

`periodSeconds`: wait 5 seconds before performing the first probe.

If the command succeeds (returns 0), the kubelet considers the container to be alive and healthy. Otherwise, the kubelet kills the container and restarts it (returns a non-zero value).

Let’s create the pod:

```sh
kubectl apply -f exec-liveness.yaml
```

Now we can describe the pod:

```sh
kubectl describe pod liveness-exec
```

```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  21s   default-scheduler  Successfully assigned default/liveness-exec to minikube
  Normal  Pulling    20s   kubelet            Pulling image "busybox"
  Normal  Pulled     17s   kubelet            Successfully pulled image "busybox" in 3.800450597s
  Normal  Created    16s   kubelet            Created container liveness
  Normal  Started    16s   kubelet            Started container livenesss
```

Wait for half a minute, then describe the pod:

```
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  66s                default-scheduler  Successfully assigned default/liveness-exec to minikube
  Normal   Pulling    65s                kubelet            Pulling image "busybox"
  Normal   Pulled     62s                kubelet            Successfully pulled image "busybox" in 3.800450597s
  Normal   Created    61s                kubelet            Created container liveness
  Normal   Started    61s                kubelet            Started container liveness
  Warning  Unhealthy  21s (x3 over 31s)  kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
  Normal   Killing    21s                kubelet            Container liveness failed liveness probe, will be restarted
```

As above, the pod is unhealthy and the kubelet will kill the container and restart it:

```
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  83s                default-scheduler  Successfully assigned default/liveness-exec to minikube
  Normal   Pulled     79s                kubelet            Successfully pulled image "busybox" in 3.800450597s
  Warning  Unhealthy  38s (x3 over 48s)  kubelet            Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
  Normal   Killing    38s                kubelet            Container liveness failed liveness probe, will be restarted
  Normal   Pulling    8s (x2 over 82s)   kubelet            Pulling image "busybox"
  Normal   Created    4s (x2 over 78s)   kubelet            Created container liveness
  Normal   Started    4s (x2 over 78s)   kubelet            Started container liveness
  Normal   Pulled     4s                 kubelet            Successfully pulled image "busybox" in 3.417375064s
```

### Configure `livenessProbe` by HTTP GET request

```yaml
# http-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/liveness
    args:
    - /server
    livenessProbe:
      httpGet: # the kubelet sends HTTP GET request
        path: /healthz
        port: 8080 # listening on port 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```

To perform a probe, the kubelet sends an HTTP GET request to the server that is running in the container and listening on port 8080. If the handler for the server’s `/healthz` path returns a success code (`[200;400)`), the kubelet considers the container to be alive and healthy. If the handler returns a failure code, the kubelet kills the container and restarts it.

```sh
kubectl apply -f http-liveness.yaml
kubectl get pods
```

```
NAME            READY   STATUS    RESTARTS     AGE
liveness-http   1/1     Running   2 (2s ago)   41s
```

After a minute, let’s describe the pod:

```sh
kubectl describe pods liveness-http
```

```
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  45s               default-scheduler  Successfully assigned default/liveness-http to minikube
  Normal   Pulled     43s               kubelet            Successfully pulled image "k8s.gcr.io/liveness" in 1.461349146s
  Normal   Pulled     23s               kubelet            Successfully pulled image "k8s.gcr.io/liveness" in 1.51144169s
  Warning  Unhealthy  6s (x6 over 30s)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    6s (x2 over 24s)  kubelet            Container liveness failed liveness probe, will be restarted
  Normal   Pulling    6s (x3 over 44s)  kubelet            Pulling image "k8s.gcr.io/liveness"
  Normal   Created    5s (x3 over 43s)  kubelet            Created container liveness
  Normal   Started    5s (x3 over 43s)  kubelet            Started container liveness
  Normal   Pulled     5s                kubelet            Successfully pulled image "k8s.gcr.io/liveness" in 1.421177595s
```

### Configure `livenessProbe`, `readinessProbe` by TCP socket

In Kubernetes, you can use `readinessProbe` to ensure that traffic does not reach a container that is not ready for it. For example, an application might need to load large data or configuration files during startup, or depend on external services after startup. In such cases, you don’t want to kill the application, but you don’t want to send it requests either. Note that `readinessProbe` runs on the container during its whole lifecycle.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-health-probes
spec:
  containers:
  - image: nginx
    name: myapp-container
    ports:
    - containerPort: 80
    readinessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 5
    livenessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 15
```

Let’s create pod:

```sh
kubectl apply -f tcp-liveness-readiness.yaml
kubectl describe pod myapp-health-probes
```

```
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  8m30s  default-scheduler  Successfully assigned default/myapp-health-probes to minikube
  Normal  Pulling    8m29s  kubelet            Pulling image "nginx"
  Normal  Pulled     8m13s  kubelet            Successfully pulled image "nginx" in 16.010486815s
  Normal  Created    8m13s  kubelet            Created container myapp-container
  Normal  Started    8m13s  kubelet            Started container myapp-container
```

### Configure `startupProbe`

Additionally, you can protect slow starting containers with `startupProbe`. Because sometimes, you have to deal with legacy applications that might require an additional startup time on their first initialization.

The trick is to set up a `startupProbe` with the same command, HTTP or TCP check, with a `failureThreshold * periodSeconds` long enough to cover the worse case startup time.

```yaml
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080

livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 1
  periodSeconds: 10

startupProbe:
  httpGet:
    path: /healthz
    port: liveness-port
  failureThreshold: 30
  periodSeconds: 10
```

As the `startupProbe` section, the application will have a maximum of 30 * 10 = 300s to finish its startup.

Once the `startupProbe` has succeeded once, the `livenessProbe` takes over to provide a fast response to container deadlocks. If the `startupProbe` never succeeds, the container is killed after 300s and subject to the pod’s `restartPolicy`.

## Managing Resource Requests and Limits

### Manage pod resource by `LimitRange`

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: example
spec:
  limits:
  - type: Pod
    min:
      cpu: 50m
      memory: 5Mi
    max:
      cpu: 1
      memory: 1Gi
  - type: Container
    defaultRequest:
      cpu: 100m
      memory: 10Mi
    default:
      cpu: 200m
      memory: 100Mi
    min:
      cpu: 50m
      memory: 5Mi
    max:
      cpu: 1
      memory: 1Gi
    maxLimitRequestRatio:
      cpu: 4
      memory: 10
  - type: PersistentVolumeClaim
    min:
      storage: 1Gi
    max:
      storage: 10Gi
```

```sh
kubectl apply -f limit-range.yaml
```

Now we try creating a pod that requests more CPU than allowed by the `LimitRange`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: big-pod
spec:
  containers:
  - image: busybox
    args: ["sleep", "9999999"]
    name: main
    resources:
      requests:
        cpu: 2
```

We will get error:

```
$ kubectl apply -f pod-with-big-resource.yaml 
The Pod "big-pod" is invalid: spec.containers[0].resources.requests: Invalid value: "2": must be less than or equal to cpu limit
```

### Manage namespace resource by `ResourceQuota`

Step 1: Create demo namespace

```sh
kubectl create namespace deployment-demo
```

Step 2: Use Resource Quotas

Create `resource-quota.yaml` file:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-quota
  namespace: deployment-demo
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 2Gi
    limits.cpu: "2"
    limits.memory: 4Gi
```

Apply to create the resource quota for deployment-demo namespace.

```sh
kubectl apply -f resource-quota.yaml
```

Now, let’s describe the deployment-demo namespace

```sh
kubectl describe namespace deployment-demo
```

```
Name:         deployment-demo
Labels:       kubernetes.io/metadata.name=deployment-demo
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:            mem-cpu-quota
  Resource         Used  Hard
  --------         ---   ---
  limits.cpu       0     2
  limits.memory    0     4Gi
  requests.cpu     0     1
  requests.memory  0     2Gi

No LimitRange resource.
```

Step 3: Create the nginx deployment

Following is `my-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: deployment-demo
  labels:
    app: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: nginx:1.20
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
      - name: logging-sidecar
        image: busybox:1.28
        command: ['sh', '-c', "while true; do echo sync logs; sleep 20; done"]
        resources:
          requests:
            memory: "32Mi"
            cpu: "125m"
          limits:
            memory: "64Mi"
            cpu: "250m"
```

Apply to create the deployment:

```sh
kubectl apply -f my-deployment.yaml
```

Now, let’s describe my deployment in deployment-demo namespace:

```sh
kubectl describe -n deployment-demo deployments.apps my-app
```

```
Name:                   my-app
Namespace:              deployment-demo
CreationTimestamp:      Tue, 05 Apr 2022 23:17:28 -0400
Labels:                 app=my-app
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=my-app
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=my-app
  Containers:
   my-app:
    Image:      nginx:1.20
    Port:       80/TCP
    Host Port:  0/TCP
    Limits:
      cpu:     500m
      memory:  128Mi
    Requests:
      cpu:        250m
      memory:     64Mi
    Environment:  <none>
    Mounts:       <none>
   logging-sidecar:
    Image:      busybox:1.28
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      while true; do echo sync logs; sleep 20; done
    Limits:
      cpu:     250m
      memory:  64Mi
    Requests:
      cpu:        125m
      memory:     32Mi
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   my-app-57d67fffc4 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  36s   deployment-controller  Scaled up replica set my-app-57d67fffc4 to 1
```

Step 4: Create the service and expose the deployment

```sh
kubectl expose deployment my-app -n deployment-demo --type=NodePort --name=my-service
kubectl get svc -n deployment-demo
```

```
kubectl get svc -n deployment-demo
NAME         TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
my-service   NodePort   10.97.225.53   <none>        80:30991/TCP   20s
```

Now, use the external IP address (`http://<minikube-ip>:<port>`) to access the nginx application:

```sh
curl http://192.168.49.2:30991
```

```
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
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

The ResourceQuota have many `.spec.hard` properties other:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-storage-quota
spec:
  hard:
    pods: 10
    replicationcontrollers: 5
    secrets: 10
    configmaps: 10
    persistentvolumeclaims: 5
    services: 5
    services.loadbalancers: 1
    services.nodeports: 2
    ssd.storageclass.storage.k8s.io/persistentvolumeclaims: 2
    requests.storage: 500Gi
    ssd.storageclass.storage.k8s.io/requests.storage: 300Gi
    standard.storageclass.storage.k8s.io/requests.storage: 1Ti
```

You can get full source code here: [job](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/job), [statefulsets](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/statefulsets), [taints-and-tolerations](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/taints-and-tolerations), [node-pod-affinity](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/node-pod-affinity), [pod-healthy-probe](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/pod-healthy-probe), [manage-resource](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/manage-resource).
