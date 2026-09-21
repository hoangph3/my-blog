---
title: "Kubernetes Security and Access Control"
date: 2023-09-23T16:14:26+07:00
tags: ["Kubernetes", "DevOps"]
author: ["hoangph3"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Harden a Kubernetes cluster with pod security policies and fine-grained RBAC for users and permissions."
summary: "Harden a Kubernetes cluster with pod security policies and fine-grained RBAC for users and permissions."
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

## Pod Security Policies

### Configure the node’s network for a pod

A pod with `hostNetwork: true` uses the node’s network interfaces instead of its own.

```yaml
# pod-host-network.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-host-network
spec:
  hostNetwork: true
  containers:
  - name: main
    image: busybox
    command: ['sh', '-c']
    args:
    - echo "$(date) Hello Kubernetes !";
      sleep 999999;
```

```sh
kubectl apply -f pod-host-network.yaml
```

```
$ kubectl exec -it pod-host-network -- sh
/ # ifconfig 
docker0   Link encap:Ethernet  HWaddr 02:42:80:85:1A:48  
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:14359 errors:0 dropped:0 overruns:0 frame:0
          TX packets:15146 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:1433351 (1.3 MiB)  TX bytes:5055820 (4.8 MiB)

eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:31:02  
          inet addr:192.168.49.2  Bcast:192.168.49.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:3718 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3598 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:1441107 (1.3 MiB)  TX bytes:1062006 (1.0 MiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:486808 errors:0 dropped:0 overruns:0 frame:0
          TX packets:486808 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:96821077 (92.3 MiB)  TX bytes:96821077 (92.3 MiB)

veth0953ee8 Link encap:Ethernet  HWaddr F2:DD:B9:2F:C6:7B  
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:84 (84.0 B)

veth8447375 Link encap:Ethernet  HWaddr 42:4F:33:37:87:C4  
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:84 (84.0 B)

...
```

We can see the eth0 ip is the control plane ip (`use: $ kubectl cluster-info`).

Without `hostNetwork: true`:

```
$ kubectl exec -it pod-host-network -- sh
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:07  
          inet addr:172.17.0.7  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

### Configure hostPort for a pod

If a host port is used, only a single pod instance can be scheduled to a node, because the port is already bound.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-host-port
spec:
  containers:
  - name: main
    image: busybox
    command: ['sh', '-c']
    args:
    - echo "$(date) Hello Kubernetes !";
      sleep 999999;
    ports:
      - containerPort: 8080
        hostPort: 9000
        protocol: TCP
```

```sh
kubectl apply -f pod-host-port.yaml
```

### Configure hostPID, hostIPC for a pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-host-pid-ipc
spec:
  hostPID: true
  hostIPC: true
  containers:
  - name: main
    image: busybox
    command: ['sh', '-c']
    args:
    - echo "$(date) Hello Kubernetes !";
      sleep 999999;
```

```sh
kubectl apply -f pod-host-pid-ipc.yaml
```

By setting the `hostIPC: true`, processes in the pod’s containers can also communicate with all the other processes running on the node, through Inter-Process Communication.

```
$ kubectl exec -it pod-host-pid-ipc -- sh
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:01 {systemd} /sbin/init
   96 root      0:00 /lib/systemd/systemd-journald
  108 108       0:00 /usr/bin/dbus-daemon --system --address=systemd: --nofork -
  114 root      0:05 /usr/bin/containerd
  121 root      0:00 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
  213 root      2:45 /usr/bin/dockerd -H tcp://0.0.0.0:2376 -H unix:///var/run/d
  805 root      5:53 /var/lib/minikube/binaries/v1.22.3/kubelet --bootstrap-kube
 1439 root      0:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 90ed40
 1440 root      0:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id ea5af7
 1444 root      0:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 1604e3
 1446 root      0:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 6d4476
 1518 65535     0:00 /pause
 1520 65535     0:00 /pause
 1528 65535     0:00 /pause
 1529 65535     0:00 /pause
 ...
```

### Running a container as a specific user

Make the container run as user guest with chown: 405.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-as-guest
spec:
  containers:
  - name: main
    image: busybox
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsUser: 405
```

```sh
kubectl apply -f pod-as-guest.yaml
```

Now we will run the id command in this new pod:

```
$ kubectl exec -it pod-as-guest -- sh
/ $ id
uid=405 gid=0(root)
/ $
```

### Preventing a container from running as root

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-as-non-root
spec:
  containers:
  - name: main
    image: busybox
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsNonRoot: true
```

```sh
kubectl apply -f pod-as-non-root.yaml
```

If you deploy this pod, it gets scheduled, but is not allowed to run:

```sh
kubectl get pods
kubectl describe pods pod-as-non-root
```

```
kubectl get pods                     
NAME              READY   STATUS                       RESTARTS   AGE
pod-as-non-root   0/1     CreateContainerConfigError   0          4m24s

Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  4m47s                  default-scheduler  Successfully assigned default/pod-as-non-root to minikube
  Normal   Pulled     3m18s                  kubelet            Successfully pulled image "busybox" in 3.064278141s
  Warning  Failed     3m1s (x8 over 4m43s)   kubelet            Error: container has runAsNonRoot and image will run as root (pod: "pod-as-non-root_default(fd120b40-998e-4421-b054-ed7284df9b00)", container: main)
  Normal   Pulled     3m1s                   kubelet            Successfully pulled image "busybox" in 3.002184008s
  Normal   Pulling    2m47s (x9 over 4m46s)  kubelet            Pulling image "busybox"
```

### Running pods in privileged mode

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-privileged
spec:
  containers:
  - name: main
    image: busybox
    command: ["/bin/sleep", "999999"]
    securityContext:
      privileged: true
```

```sh
kubectl apply -f pod-privileged.yaml
```

We can list the device files your privileged pod. In fact, the privileged container sees
all the host node’s devices. This means it can use any device freely.

```
$ kubectl exec -it pod-privileged -- sh
/ # ls /dev
autofs           kmsg             rtc0             tty13            tty30            tty48            tty8             vcs6             vcsu7
bsg              kvm              sda              tty14            tty31            tty49            tty9             vcs7             vcsu8
btrfs-control    loop-control     sda1             tty15            tty32            tty5             ttyS0            vcs8             vfio
bus              mapper           sda2             tty16            tty33            tty50            ttyS1            vcsa             vga_arbiter
core             media0           sg0              tty17            tty34            tty51            ttyS2            vcsa1            vhci
cpu              mei0             shm              tty18            tty35            tty52            ttyS3            vcsa2            vhost-net
cpu_dma_latency  mem              snapshot         tty19            tty36            tty53            uhid             vcsa3            vhost-vsock
cuse             mqueue           snd              tty2             tty37            tty54            uinput           vcsa4            video0
dri              net              stderr           tty20            tty38            tty55            urandom          vcsa5            video1
drm_dp_aux0      null             stdin            tty21            tty39            tty56            vboxdrv          vcsa6            watchdog
drm_dp_aux1      nvram            stdout           tty22            tty4             tty57            vboxdrvu         vcsa7            watchdog0
drm_dp_aux2      port             termination-log  tty23            tty40            tty58            vboxnetctl       vcsa8            watchdog1
fb0              ppp              tpm0             tty24            tty41            tty59            vboxusb          vcsu             zero
fd               psaux            tty              tty25            tty42            tty6             vcs              vcsu1
full             ptmx             tty0             tty26            tty43            tty60            vcs1             vcsu2
fuse             ptp0             tty1             tty27            tty44            tty61            vcs2             vcsu3
hidraw0          pts              tty10            tty28            tty45            tty62            vcs3             vcsu4
hpet             random           tty11            tty29            tty46            tty63            vcs4             vcsu5
input            rfkill           tty12            tty3             tty47            tty7             vcs5             vcsu6
```

### Dropping capabilities from a container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-drop-chown-capability
spec:
  containers:
  - name: main
    image: busybox
    command: ["/bin/sleep", "999999"]
    securityContext:
      capabilities:
        drop:
        - CHOWN
```

```sh
kubectl apply -f pod-drop-chown-capability.yaml
```

By dropping the CHOWN capability, you’re not allowed to change the owner of the /tmp directory in this pod:

```
$ kubectl exec -it pod-drop-chown-capability -- sh
/ # chown 1000:1000 /tmp/
chown: /tmp/: Operation not permitted
```

### Preventing processes from writing to the container’s filesystem

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-readonly-filesystem
spec:
  containers:
  - name: main
    image: busybox
    command: ["/bin/sleep", "999999"]
    securityContext:
      readOnlyRootFilesystem: true
    volumeMounts:
    - name: my-volume
      mountPath: /volume
      readOnly: false
  volumes:
  - name: my-volume
    emptyDir:
```

```sh
kubectl apply -f pod-with-readonly-filesystem.yaml
```

When you deploy this pod, the container is running as root, which has write permissions to the / directory, but trying to write a file there fails:

```
$ kubectl exec -it pod-with-readonly-filesystem -- sh 
/ # touch a.txt
touch: a.txt: Read-only file system
```

### Sharing volumes when containers run as different users

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-shared-volume
spec:
  containers:
  - name: first
    image: busybox
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsUser: 1111
    volumeMounts:
    - name: shared-volume
      mountPath: /volume
      readOnly: false
  - name: second
    image: busybox
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsUser: 2222
    volumeMounts:
    - name: shared-volume
      mountPath: /volume
      readOnly: false
  volumes:
  - name: shared-volume
    emptyDir:
```

The first container runs as user ID 1111, the second container runs as user ID 2222, and both containers use the same volume.

```sh
kubectl apply -f pod-with-shared-volume.yaml
```

```
$ kubectl exec -it pod-with-shared-volume -c first -- sh
/ $ ls -la | grep volume
drwxrwxrwx    2 root     root          4096 Apr 20 16:37 volume
/ $ echo foo > volume/foo 
/ $ ls -la volume/
total 12
drwxrwxrwx    2 root     root          4096 Apr 20 16:37 .
drwxr-xr-x    1 root     root          4096 Apr 20 16:32 ..
-rw-r--r--    1 1111     root             4 Apr 20 16:39 foo
```

As you can see, the group ID is always root, the container runs as user ID that we set in `.spec.securityContext.runAsUser`.

What happen when we define the `fsGroup` and `supplementalGroups` in the security context at the pod level?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-shared-volume-fsgroup
spec:
  securityContext:
    fsGroup: 555
    supplementalGroups: [666, 777]
  containers:
  - name: first
    image: busybox
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsUser: 1111
    volumeMounts:
    - name: shared-volume
      mountPath: /volume
      readOnly: false
  - name: second
    image: busybox
    command: ["/bin/sleep", "999999"]
    securityContext:
      runAsUser: 2222
    volumeMounts:
    - name: shared-volume
      mountPath: /volume
      readOnly: false
  volumes:
  - name: shared-volume
    emptyDir:
```

```sh
kubectl apply -f pod-with-shared-volume-fsgroup.yaml
```

```
$ kubectl exec -it pod-with-shared-volume-fsgroup -c first -- sh
/ $ id
uid=1111 gid=0(root) groups=555,666,777
/ $ ls -la | grep volume
drwxrwsrwx    2 root     555           4096 Apr 20 16:42 volume
/ $ echo bar > volume/bar
/ $ ls -la volume/
total 12
drwxrwsrwx    2 root     555           4096 Apr 20 16:43 .
drwxr-xr-x    1 root     root          4096 Apr 20 16:42 ..
-rw-r--r--    1 1111     555              4 Apr 20 16:43 bar
/ $ echo foo > tmp/foo
/ $ ls -la tmp/
total 12
drwxrwxrwt    1 root     root          4096 Apr 20 16:43 .
drwxr-xr-x    1 root     root          4096 Apr 20 16:42 ..
-rw-r--r--    1 1111     root             4 Apr 20 16:43 foo
```

In the pod definition, we set `fsGroup` to `555`, so the mounted volume will be owned by group ID `555`. And because we create a file in the mounted volume’s directory, the file is owned by user ID `1111` and by group ID `555`.

### Enabling network isolation in a namespace

By default, pods in a given namespace can be accessed by anyone. Now we can use `NetworkPolicy` to prevent all clients from connecting to any pod in your namespace.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector:
```

Note that: Empty pod selector matches all pods in the same namespace.

To let clients connect to the pods in the namespace, you must now explicitly say who can connect to the pods.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-netpolicy
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: webserver
    ports:
    - port: 5432
```

The example `NetworkPolicy` allows pods with the `app=webserver` label to connect to pods with the `app=database` label, only on port `5432`. Other pods can’t connect to the database pods, and no one (not even the webserver pods) can connect to anything other than port `5432` of the database pods.

A `NetworkPolicy` only allowing pods in namespaces matching a `namespaceSelector` to access a specific pod.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: shoppingcart-netpolicy
spec:
  podSelector:
    matchLabels:
      app: shopping-cart 
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tenant: manning
    ports:
    - port: 80
```

You can also specify an IP block in CIDR notation.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ipblock-netpolicy
spec:
  podSelector:
    matchLabels:
      app: shopping-cart
  ingress:
  - from:
    - ipBlock:
        cidr: 192.168.1.0/24
```

You can also limit their outbound traffic through egress rules.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-net-policy
spec:
  podSelector:
    matchLabels:
      app: webserver
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - port: 5432
```

## Users and Permissions

### Authentication and Authorization in Kubernetes

In this tutorial, we will create two namespaces “my-app-dev” and “my-app-prod” and two users “dev” and “test” with different roles to those namespaces:

```
my-app-dev:
  dev: Edit
my-app-prod:
  dev: View
  test: Edit
```

The default Roles defined in Kubernetes are:

```
- view: read-only access, excludes secrets.
- edit: above + ability to edit most resources, excludes roles and role bindings.
- admin: above + ability to manage roles and role bindings at a namespace level.
- cluster-admin: everything.
```

To manage normal users, we will use X.509 client certificate:

```
1. Create a user's private key and a certificate signing request.
2. Get it certified by a CA (Kubernetes CA) to have the user's certificate.
```

Suppose you are the admin (like root privileges), now you will create the normal user `dev:123456`:

```sh
sudo useradd -m dev
sudo passwd dev
```

```
New password: 123456
Retype new password: 123456
passwd: password updated successfully
```

Go into dev’s home directory as root privileges:

```sh
sudo su
cd /home/dev && ls -la
```

```
┌──(root💀jump-windows)-[/home/dev]
└─# cd /home/dev && ls -la
total 60
drwxr-xr-x 4 dev  dev   4096 Apr 11 13:09 .
drwxr-xr-x 4 root root  4096 Apr 11 13:09 ..
-rw-r--r-- 1 dev  dev    220 Feb 24  2021 .bash_logout
-rw-r--r-- 1 dev  dev   5551 Apr  2 00:54 .bashrc
-rw-r--r-- 1 dev  dev   3526 Feb 24  2021 .bashrc.original
drwxr-xr-x 5 dev  dev   4096 Jan 24 05:06 .config
-rw-r--r-- 1 dev  dev  11759 May 19  2021 .face
lrwxrwxrwx 1 dev  dev      5 Mar  4 11:38 .face.icon -> .face
drwxr-xr-x 3 dev  dev   4096 Jan 24 05:06 .java
-rw-r--r-- 1 dev  dev    807 Feb 24  2021 .profile
-rw-r--r-- 1 dev  dev  10875 Feb  3 22:45 .zshrc
```

Create a private key:

```sh
openssl genrsa -out dev.key 2048
```

Create a certificate signing request (CSR):

```sh
openssl req -new -key dev.key -subj "/CN=dev" -out dev.csr
```

Sign the CSR with the Kubernetes CA:

If minikube:

```sh
openssl x509 -req -in dev.csr -CA /home/ph3/.minikube/ca.crt -CAkey /home/ph3/.minikube/ca.key -CAcreateserial -out dev.crt -days 500
```

If kubernetes cluster:

```sh
openssl x509 -req -in dev.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -out dev.crt -days 500
```

List files in dev’s home directory:

```
┌──(root💀jump-windows)-[/home/dev]
└─# ls -la
total 72
drwxr-xr-x 4 dev  dev   4096 Apr 11 13:17 .
drwxr-xr-x 4 root root  4096 Apr 11 13:09 ..
-rw-r--r-- 1 dev  dev    220 Feb 24  2021 .bash_logout
-rw-r--r-- 1 dev  dev   5551 Apr  2 00:54 .bashrc
-rw-r--r-- 1 dev  dev   3526 Feb 24  2021 .bashrc.original
drwxr-xr-x 5 dev  dev   4096 Jan 24 05:06 .config
-rw-r--r-- 1 root root   985 Apr 11 13:17 dev.crt
-rw-r--r-- 1 root root   883 Apr 11 13:13 dev.csr
-rw------- 1 root root  1675 Apr 11 13:12 dev.key
-rw-r--r-- 1 dev  dev  11759 May 19  2021 .face
lrwxrwxrwx 1 dev  dev      5 Mar  4 11:38 .face.icon -> .face
drwxr-xr-x 3 dev  dev   4096 Jan 24 05:06 .java
-rw-r--r-- 1 dev  dev    807 Feb 24  2021 .profile
-rw-r--r-- 1 dev  dev  10875 Feb  3 22:45 .zshrc
```

We have new three files: dev.crt, dev.csr, dev.key but under root privileges. We need to grant all the created files and directories to the “dev” user:

```sh
chown -R dev: /home/dev/
```

```
┌──(root💀jump-windows)-[/home/dev]
└─# ls -la
total 72
drwxr-xr-x 4 dev  dev   4096 Apr 11 13:17 .
drwxr-xr-x 4 root root  4096 Apr 11 13:09 ..
-rw-r--r-- 1 dev  dev    220 Feb 24  2021 .bash_logout
-rw-r--r-- 1 dev  dev   5551 Apr  2 00:54 .bashrc
-rw-r--r-- 1 dev  dev   3526 Feb 24  2021 .bashrc.original
drwxr-xr-x 5 dev  dev   4096 Jan 24 05:06 .config
-rw-r--r-- 1 dev  dev    985 Apr 11 13:17 dev.crt
-rw-r--r-- 1 dev  dev    883 Apr 11 13:13 dev.csr
-rw------- 1 dev  dev   1675 Apr 11 13:12 dev.key
-rw-r--r-- 1 dev  dev  11759 May 19  2021 .face
lrwxrwxrwx 1 dev  dev      5 Mar  4 11:38 .face.icon -> .face
drwxr-xr-x 3 dev  dev   4096 Jan 24 05:06 .java
-rw-r--r-- 1 dev  dev    807 Feb 24  2021 .profile
-rw-r--r-- 1 dev  dev  10875 Feb  3 22:45 .zshrc
```

Now we will login to “dev” user to create user and context in kubernetes:

```sh
sudo -u dev bash
```

Create the user inside kubernetes as “dev” privileges:

```sh
kubectl config set-credentials dev --client-certificate=/home/dev/dev.crt --client-key=/home/dev/dev.key
```

```
┌──(dev㉿jump-windows)-[~]
└─$ kubectl config set-credentials dev --client-certificate=/home/dev/dev.crt --client-key=/home/dev/dev.key
User "dev" set.
```

Create a context for the user:

If minikube:

```sh
kubectl config set-context dev-context --cluster=minikube --user=dev
```

If kubernetes cluster:

```sh
kubectl config set-context dev-context --cluster=kubernetes --user=dev
```

After above steps, the `/home/dev/.kube/config` file is created:

```yaml
apiVersion: v1
clusters: null
contexts:
- context:
    cluster: minikube
    user: dev
  name: dev-context
current-context: ""
kind: Config
preferences: {}
users:
- name: dev
  user:
    client-certificate: /home/dev/dev.crt
    client-key: /home/dev/dev.key
```

Edit user config file:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/ph3/.minikube/ca.crt
    server: https://192.168.49.2:8443 
  name: minikube
contexts:
- context:
    cluster: minikube
    user: dev
  name: dev-context
current-context: dev-context
kind: Config
preferences: {}
users:
- name: dev
  user:
    client-certificate: /home/dev/dev.crt
    client-key: /home/dev/dev.key
```

Note that the `certificate-authority` and `server` variables have to be as in the cluster admin config.

Now we have a user “dev” created. We will do the same for user “test”.

As administrator, we can create the two namespaces:

```sh
kubectl create namespace my-app-dev
kubectl create namespace my-app-prod
```

As we have not defined any authorization to the users, they should get forbidden access to all cluster resources.

```
┌──(dev㉿jump-windows)-[~]
└─$ kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "dev" cannot list resource "nodes" in API group "" at the cluster scope

┌──(dev㉿jump-windows)-[~]
└─$ kubectl get pods
Error from server (Forbidden): pods is forbidden: User "dev" cannot list resource "pods" in API group "" in the namespace "default"

┌──(dev㉿jump-windows)-[~]
└─$ kubectl get pods -n my-app-dev
Error from server (Forbidden): pods is forbidden: User "dev" cannot list resource "pods" in API group "" in the namespace "my-app-dev"

┌──(dev㉿jump-windows)-[~]
└─$ kubectl get pods -n my-app-prod
Error from server (Forbidden): pods is forbidden: User "dev" cannot list resource "pods" in API group "" in the namespace "my-app-prod"
```

```
┌──(test㉿jump-windows)-[~]
└─$ kubectl get nodes
Error from server (Forbidden): nodes is forbidden: User "test" cannot list resource "nodes" in API group "" at the cluster scope

┌──(test㉿jump-windows)-[~]
└─$ kubectl get pods
Error from server (Forbidden): pods is forbidden: User "test" cannot list resource "pods" in API group "" in the namespace "default"

┌──(test㉿jump-windows)-[~]
└─$ kubectl get pods -n my-app-dev
Error from server (Forbidden): pods is forbidden: User "test" cannot list resource "pods" in API group "" in the namespace "my-app-dev"

┌──(test㉿jump-windows)-[~]
└─$ kubectl get pods -n my-app-prod
Error from server (Forbidden): pods is forbidden: User "test" cannot list resource "pods" in API group "" in the namespace "my-app-prod"
```

As administrator, we will create a Role/ClusterRole. A Role/ClusterRole are just a list of verbs (actions) permitted on specific resources and namespaces. Here is `role.yaml` file:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: list-deployments
  namespace: my-app-dev
rules:
  - apiGroups: [ apps ]
    resources: [ deployments ]
    verbs: [ get, list ]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: list-deployments
rules:
  - apiGroups: [ apps ]
    resources: [ deployments ]
    verbs: [ get, list ]
```

```sh
kubectl apply -f role.yaml
```

We can list ClusterRole:

```sh
kubectl get clusterrole
```

```
NAME                                                                   CREATED AT
admin                                                                  2022-01-27T01:18:51Z
cluster-admin                                                          2022-01-27T01:18:51Z
edit                                                                   2022-01-27T01:18:51Z
ingress-nginx                                                          2022-04-10T14:38:31Z
ingress-nginx-admission                                                2022-04-10T14:38:31Z
kubeadm:get-nodes                                                      2022-01-27T01:18:53Z
list-deployments                                                       2022-04-11T18:18:04Z
system:aggregate-to-admin                                              2022-01-27T01:18:51Z
system:aggregate-to-edit                                               2022-01-27T01:18:51Z
system:aggregate-to-view                                               2022-01-27T01:18:51Z
system:auth-delegator                                                  2022-01-27T01:18:51Z
system:basic-user                                                      2022-01-27T01:18:51Z
...
...
system:volume-scheduler                                                2022-01-27T01:18:51Z
view                                                                   2022-01-27T01:18:51Z
```

We are now going to bind Role/ClusterRole to users by `role-binding.yaml` as below:

```
dev:
  edit on namespace "my-app-dev"
  view on namespace "my-app-prod"
test:
  edit on namespace "my-app-prod"
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev
  namespace: my-app-dev
subjects:
- kind: User
  name: dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev
  namespace: my-app-prod
subjects:
- kind: User
  name: dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: test
  namespace: my-app-prod
subjects:
- kind: User
  name: test
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

```sh
kubectl apply -f role-binding.yaml
```

Let’s check if our users have the right permissions.

```
User: test (edit on "my-app-prod" namespace)
  - can create deployments, list pods on "my-app-prod"
  - can't create or list pods, deployments on "my-app-dev"
```

```
┌──(test㉿jump-windows)-[~]
└─$ kubectl run nginx --image=nginx -n my-app-prod
pod/nginx created

┌──(test㉿jump-windows)-[~]
└─$ kubectl get pods -n my-app-prod
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          15s

┌──(test㉿jump-windows)-[~]
└─$ kubectl run nginx --image=nginx -n my-app-dev
Error from server (Forbidden): pods is forbidden: User "test" cannot create resource "pods" in API group "" in the namespace "my-app-dev"

┌──(test㉿jump-windows)-[~]
└─$ kubectl get pods -n my-app-dev
Error from server (Forbidden): pods is forbidden: User "test" cannot list resource "pods" in API group "" in the namespace "my-app-dev"
```

```
User: dev (view on "my-app-prod" namespace
           edit on "my-app-dev" namespace)
  - can list pods, deployments on "my-app-prod" but can't create
  - can list, create, delete pods, deployments on "my-app-dev"
```

```
┌──(dev㉿jump-windows)-[~]
└─$ kubectl get pods -n my-app-prod
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          7m14s

┌──(dev㉿jump-windows)-[~]
└─$ kubectl run nginx --image=nginx -n my-app-prod
Error from server (Forbidden): pods is forbidden: User "dev" cannot create resource "pods" in API group "" in the namespace "my-app-prod"

┌──(dev㉿jump-windows)-[~]
└─$ kubectl run nginx --image=nginx -n my-app-dev
pod/nginx created

┌──(dev㉿jump-windows)-[~]
└─$ kubectl get pods -n my-app-dev
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          11s

┌──(dev㉿jump-windows)-[~]
└─$ kubectl delete pods nginx -n my-app-dev
pod "nginx" deleted

┌──(dev㉿jump-windows)-[~]
└─$ kubectl get pods -n my-app-dev
No resources found in my-app-dev namespace.
```

### Configure ServiceAccount to pull private image

Create service account with imagePullSecrets by `sa-image-pull-secrets.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-docker-registry
imagePullSecrets:
- name: mydockerhubsecret
```

Now, you can inspect the ServiceAccount with the describe command:

```sh
kubectl describe sa sa-docker-registry
```

```
Name:                sa-docker-registry
Namespace:           default
Labels:              <none>
Annotations:         <none>
Image pull secrets:  mydockerhubsecret
Mountable secrets:   sa-docker-registry-token-bcbmg
Tokens:              sa-docker-registry-token-bcbmg
Events:              <none>
```

You can see that a custom token Secret has been created and associated with the ServiceAccount.

Let’s inspect the secret `sa-docker-registry-token-bcbmg`:

```sh
kubectl describe secret sa-docker-registry-token-bcbmg
```

```
Name:         sa-docker-registry-token-bcbmg
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: sa-docker-registry
              kubernetes.io/service-account.uid: 643bb848-6f9f-4f4e-a087-0ce61fec527f

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1111 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSUzI1NiIs...
```

To assign a ServiceAccount to a pod, we will set the name of the ServiceAccount in the `spec.serviceAccountName` field in the pod definition:

This is `busybox-sa.yaml` without service account:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
  - name: busybox
    image: hoangph3/busybox
    command: ['sh', '-c']
    args: 
    - echo "$(date) Hello Kubernetes !";
      sleep 9999999;
```

```sh
kubectl apply -f busybox-sa.yaml
kubectl get pods
kubectl describe pod busybox
```

```
NAME      READY   STATUS         RESTARTS   AGE
busybox   0/1     ErrImagePull   0          7s

Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Normal   Scheduled  16s   default-scheduler  Successfully assigned default/busybox to minikube
  Normal   Pulling    15s   kubelet            Pulling image "hoangph3/busybox"
  Warning  Failed     11s   kubelet            Failed to pull image "hoangph3/busybox": rpc error: code = Unknown desc = Error response from daemon: pull access denied for hoangph3/busybox, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed     11s   kubelet            Error: ErrImagePull
  Normal   BackOff    11s   kubelet            Back-off pulling image "hoangph3/busybox"
  Warning  Failed     11s   kubelet            Error: ImagePullBackOff
```

This is `busybox-sa.yaml` with service account:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  serviceAccountName: sa-docker-registry
  containers:
  - name: busybox
    image: hoangph3/busybox
    command: ['sh', '-c']
    args: 
    - echo "$(date) Hello Kubernetes !";
      sleep 9999999;
```

```sh
kubectl apply -f busybox-sa.yaml
kubectl get pods
kubectl describe pod busybox
kubectl logs -f busybox
```

```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          20s

Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  15s   default-scheduler  Successfully assigned default/busybox to minikube
  Normal  Pulling    14s   kubelet            Pulling image "hoangph3/busybox"
  Normal  Pulled     10s   kubelet            Successfully pulled image "hoangph3/busybox" in 3.227788488s
  Normal  Created    10s   kubelet            Created container busybox
  Normal  Started    10s   kubelet            Started container busybox

Wed Apr 13 16:31:10 UTC 2022 Hello Kubernetes !
```

The custom ServiceAccount’s token is mounted into the container, you can confirm by exec command:

```
$ kubectl exec -it busybox -- sh                                              
/ # cd /var/run/secrets/kubernetes.io/serviceaccount/
/var/run/secrets/kubernetes.io/serviceaccount # ls
ca.crt     namespace  token
/var/run/secrets/kubernetes.io/serviceaccount # cat token 
eyJhbGciOiJSUzI1NiIs...
```

### Connect to cluster with ServiceAccount Token

When we create the service account, we will get the CA certificate, namespace, and token. We will use this information to connect to cluster.

First build a docker image with Dockerfile:

```sh
FROM python:3.8-slim-buster
RUN apt -y update && apt -y install curl telnet nano && curl -L -O https://dl.k8s.io/v1.23.5/kubernetes-client-linux-amd64.tar.gz && tar zvxf kubernetes-client-linux-amd64.tar.gz kubernetes/client/bin/kubectl && mv kubernetes/client/bin/kubectl / && rm -rf kubernetes && rm -f kubernetes-client-linux-amd64.tar.gz
CMD ["sleep", "9999999"]
```

Build image:

```sh
docker build -t hoangph3/kubectl-client .
```

```
$ docker images

REPOSITORY                                TAG               IMAGE ID       CREATED          SIZE
hoangph3/kubectl-client                   latest            535bed2a335c   6 seconds ago    185MB
hoangph3/kubectl-proxy                    latest            9a536cb8b202   36 minutes ago   184MB
redis                                     latest            bba24acba395   2 weeks ago      113MB
nginx                                     latest            12766a6745ee   2 weeks ago      142MB
curlimages/curl                           latest            375c62ad3696   2 weeks ago      8.3MB
```

Now, we are going to create one pod in namespace `foo` and the other one in namespace `bar`:

```sh
$ kubectl create ns foo 
namespace/foo created

$ kubectl run test --image=hoangph3/kubectl-client --image-pull-policy=Never -n foo
pod/test created

$ kubectl create ns bar
namespace/bar created

$ kubectl run test --image=hoangph3/kubectl-client --image-pull-policy=Never -n bar 
pod/test created
```

Now we will use `kubectl exec` to run a shell inside each of the two pods (one in each terminal):

```sh
kubectl exec -it test -n foo -- bash

root@test:/# env
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_PORT=443
HOSTNAME=test
PYTHON_VERSION=3.8.12
PWD=/
PYTHON_SETUPTOOLS_VERSION=57.5.0
HOME=/root
LANG=C.UTF-8
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
GPG_KEY=E3FF2839C048B25C084DEBE9B26995E310250568
TERM=xterm
SHLVL=1
KUBERNETES_PORT_443_TCP_PROTO=tcp
PYTHON_PIP_VERSION=21.2.4
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PYTHON_GET_PIP_SHA256=7c5239cea323cadae36083079a5ee6b2b3d56f25762a0c060d2867b89e5e06c5
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
PYTHON_GET_PIP_URL=https://github.com/pypa/get-pip/raw/2caf84b14febcda8077e59e9b8a6ef9a680aa392/public/get-pip.py
PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
_=/usr/bin/env

root@test:/# /kubectl get svc
Error from server (Forbidden): services is forbidden: User "system:serviceaccount:foo:default" cannot list resource "services" in API group "" in the namespace "foo"
```

Because the default permissions for a ServiceAccount don’t allow it to list or modify any resources. Now, let’s learn how to allow the ServiceAccount to do that. First, you’ll need to create a Role resource.

```sh
kubectl create role service-reader --verb=get --verb=list --resource=services -n foo
role.rbac.authorization.k8s.io/service-reader created

kubectl create role service-reader --verb=get --verb=list --resource=services -n bar
role.rbac.authorization.k8s.io/service-reader created
```

A Role defines what actions can be performed, but it doesn’t specify who can perform them. To do that, you must bind the Role to a subject, which can be a user, a Service-Account, or a group (of users or ServiceAccounts).

Binding Roles to subjects is achieved by creating a RoleBinding resource. To bind the Role to the default ServiceAccount, run the following command:

```sh
kubectl create rolebinding test --role=service-reader --serviceaccount=foo:default -n foo
rolebinding.rbac.authorization.k8s.io/test created
```

Note that, if you want to bind a Role to a user instead of a ServiceAccount, use the `--user` argument to specify the username. To bind it to a group, use `--group`.

Because this RoleBinding binds the Role to the ServiceAccount the pod in namespace foo is running under, you can now list Services from within that pod.

```sh
kubectl exec -it test -n foo -- bash

root@test:/# /kubectl get svc
No resources found in foo namespace.
```

Because we only create the RoleBinding for the foo namespace, the bar namespace is not. This leads the pod in namespace bar can’t list the Services in its own namespace, and obviouslyalso not those in the foo namespace.

```sh
kubectl exec -it test -n bar -- bash

root@test:/# /kubectl get svc
Error from server (Forbidden): services is forbidden: User "system:serviceaccount:bar:default" cannot list resource "services" in API group "" in the namespace "bar"

root@test:/# /kubectl get svc -n foo
Error from server (Forbidden): services is forbidden: User "system:serviceaccount:bar:default" cannot list resource "services" in API group "" in the namespace "foo"
```

But you can edit your RoleBinding in the foo namespace and add the other pod’s ServiceAccount, even though it’s in a different namespace. Run the following command:

```sh
kubectl edit rolebinding test -n foo

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2022-04-16T06:36:58Z"
  name: test
  namespace: foo
  resourceVersion: "376922"
  uid: b35fb7a1-82d0-4114-a4a6-2602ec00394f
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: service-reader
subjects:
- kind: ServiceAccount
  name: default
  namespace: foo
```

Then add the following lines to the list of subjects, look like that:

```
...
subjects:
- kind: ServiceAccount
  name: default
  namespace: foo
- kind: ServiceAccount
  name: default
  namespace: bar
```

Now you can also list Services in the `foo` namespace from inside the pod running in the `bar` namespace:

```sh
kubectl exec -it test -n bar -- bash

root@test:/# /kubectl get svc
Error from server (Forbidden): services is forbidden: User "system:serviceaccount:bar:default" cannot list resource "services" in API group "" in the namespace "bar"

root@test:/# /kubectl get svc -n foo
No resources found in foo namespace.
```

### ClusterRoles and ClusterRoleBindings

A ClusterRole can be used to allow access to cluster-level resources. Let’s look at how to allow your pod to list PersistentVolumes in your cluster. First, we will create a ClusterRole called `pv-reader`:

```sh
kubectl create clusterrole pv-reader --verb=get,list --resource=persistentvolumes
clusterrole.rbac.authorization.k8s.io/pv-reader created
```

We will exec to the pod in `foo` namespace and list PersistentVolumes:

```sh
kubectl exec -it test -n foo -- bash

root@test:/# /kubectl get pv
Error from server (Forbidden): persistentvolumes is forbidden: User "system:serviceaccount:foo:default" cannot list resource "persistentvolumes" in API group "" at the cluster scope
```

Default ServiceAccount is unable to get and list PersistentVolumes, you must bind the ClusterRole to a ServiceAccount by run the following command:

```sh
kubectl create clusterrolebinding pv-test --clusterrole=pv-reader --serviceaccount=foo:default
clusterrolebinding.rbac.authorization.k8s.io/pv-test created
```

Let’s see if you can list PersistentVolumes now:

```sh
kubectl exec -it test -n foo -- bash

root@test:/# /kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                  STORAGECLASS   REASON   AGE
pvc-5865deec-56a1-4743-9e9f-e638b9e959fa   1Mi        RWO            Delete           Bound    default/data-kubia-1   standard                5d
pvc-d74affe3-ded5-46f7-889d-c0abf4c12e0a   1Mi        RWO            Delete           Bound    default/data-kubia-0   standard                5d
```

Note that if you create a `ClusterRoleBinding` and reference the `ClusterRole` in it, the subjects listed in the binding can view the specified resources across all namespaces. If, on the other hand, you create a `RoleBinding`, the subjects listed in the binding can only view resources in the namespace of the `RoleBinding`. We will confirm now.

First, trying to list pods across all namespaces and `foo` namespace:

```sh
kubectl exec -it test -n foo -- bash

root@test:/# /kubectl get pods --all-namespaces
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:foo:default" cannot list resource "pods" in API group "" at the cluster scope

root@test:/# /kubectl get pods -n foo
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:foo:default" cannot list resource "pods" in API group "" in the namespace "foo"
```

Now, let’s see what happens when you create a ClusterRoleBinding (not RoleBinding) and bind it to the pod’s ServiceAccount:

```sh
kubectl create clusterrolebinding view-test --clusterrole=view --serviceaccount=foo:default
clusterrolebinding.rbac.authorization.k8s.io/view-test created

kubectl exec -it test -n foo -- bash
root@test:/# /kubectl get pods --all-namespaces
NAMESPACE       NAME                                        READY   STATUS      RESTARTS         AGE
bar             test                                        1/1     Running     3 (6h21m ago)    39h
foo             test                                        1/1     Running     3 (6h21m ago)    39h
ingress-nginx   ingress-nginx-admission-create--1-6rs7v     0/1     Completed   0                5d17h
ingress-nginx   ingress-nginx-admission-patch--1-qf2zd      0/1     Completed   1                5d17h
ingress-nginx   ingress-nginx-controller-5f66978484-wtpph   1/1     Running     7 (6h21m ago)    5d17h
kube-system     coredns-78fcd69978-kfxrb                    1/1     Running     29 (8h ago)      79d
kube-system     etcd-minikube                               1/1     Running     29 (8h ago)      79d
kube-system     kube-apiserver-minikube                     1/1     Running     29 (8h ago)      79d
kube-system     kube-controller-manager-minikube            1/1     Running     30 (8h ago)      79d
kube-system     kube-proxy-wjfqr                            1/1     Running     29 (6h21m ago)   79d
kube-system     kube-scheduler-minikube                     1/1     Running     29 (6h21m ago)   79d
kube-system     storage-provisioner                         1/1     Running     59 (6h19m ago)   79d
my-app-prod     nginx                                       1/1     Running     6 (6h21m ago)    4d13h

root@test:/# /kubectl get pods -n foo
NAME   READY   STATUS    RESTARTS        AGE
test   1/1     Running   3 (6h22m ago)   39h

root@test:/# /kubectl get pods -n bar
NAME   READY   STATUS    RESTARTS        AGE
test   1/1     Running   3 (6h22m ago)   39h
```

As expected, the pod can get a list of all the pods in the cluster. Let’s describe the view-test ClusterRole:

```sh
kubectl get clusterrolebindings.rbac.authorization.k8s.io view-test -o yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: "2022-04-16T08:30:33Z"
  name: view-test
  resourceVersion: "382606"
  uid: 08b8d16a-8bc7-4b95-8ba3-c28650b08f9f
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
- kind: ServiceAccount
  name: default
  namespace: foo
```

To summarize, combining a ClusterRoleBinding with a ClusterRole referring to namespaced resources allows the pod to access namespaced resources in any namespace.

Now we will use RoleBinding instead ClusterRoleBinding:

```sh
kubectl delete clusterrolebindings.rbac.authorization.k8s.io view-test
clusterrolebinding.rbac.authorization.k8s.io "view-test" deleted

kubectl create rolebinding view-test --clusterrole=view --serviceaccount=foo:default -n foo 
rolebinding.rbac.authorization.k8s.io/view-test created

kubectl exec -it test -n foo -- bash
root@test:/# /kubectl get pods --all-namespaces
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:foo:default" cannot list resource "pods" in API group "" at the cluster scope

root@test:/# /kubectl get pods -n bar
Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:foo:default" cannot list resource "pods" in API group "" in the namespace "bar"

root@test:/# /kubectl get pods -n foo
NAME   READY   STATUS    RESTARTS        AGE
test   1/1     Running   3 (6h33m ago)   39h
```

It’s not too surprising, your pod can list pods in the `foo` namespace, but not in any other specific namespace or across all namespaces.

You can get full source code here: [security-policy](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/security-policy), [users-and-permissions](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/components/users-and-permissions).
