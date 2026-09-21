---
title: "Deploying Demo Applications on Kubernetes"
date: 2023-09-23T16:14:26+07:00
tags: ["Kubernetes", "DevOps"]
author: ["hoangph3"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Two end-to-end example apps deployed on Kubernetes: a Node.js + MongoDB app and a Python + MySQL app."
summary: "Two end-to-end example apps deployed on Kubernetes: a Node.js + MongoDB app and a Python + MySQL app."
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

## Node.js + MongoDB Demo

## demo app - developing with Kubernetes

This demo app shows a simple user profile app set up using

- server.js run web app
- mongo for data storage

All components are kubernetes

### With minikube

#### To start the application

Step 1: Moving docker environment to kubernetes

```
eval $(minikube docker-env)
```

Step 2: Build python app image

```
docker build -t mongo-demo-k8s:1.0 .
```

Step 3: Create deployment and service

```
kubectl apply -f mongo-config.yaml
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo.yaml
kubectl apply -f webapp.yaml
```

Step 4: Get public IP of minikube

```
minikube ip #192.168.49.2
```

Step 5: Access you python application UI from browser

```
curl http://192.168.49.2:30100
```

## Python + MySQL Demo

## demo app - developing with Kubernetes

This demo app shows a simple user profile app set up using

- main.py run web app
- mysql for data storage

All components are kubernetes

### With minikube

#### To start the application

Step 1: Moving docker environment to kubernetes

```
eval $(minikube docker-env)
```

Step 2: Build python app image

```
docker build -t python-docker-dev .
```

Step 3: Create deployment and service

```
kubectl apply -f mysql.yaml
kubectl apply -f webapp.yaml
```

Step 4: Get public IP of minikube

```
minikube ip #192.168.49.2
```

Step 5: Access you python application UI from browser

```
curl http://192.168.49.2:30900
curl http://192.168.49.2:30900/initdb
curl http://192.168.49.2:30900/widgets
```

You can get full source code here: [node-mongo](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/demo/node-mongo), [python-mysql](https://github.com/hoangph3/devops-tutorial/tree/main/kubernetes/demo/python-mysql).
