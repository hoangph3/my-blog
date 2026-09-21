---
title: "Deploying Your First Model with Seldon Core"
date: 2022-12-01T21:01:41+07:00
tags: ["Seldon Core", "Kubernetes", "Model Serving", "MLOps"]
author: ["hoangph3"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "Deploy a first machine learning model on Kubernetes using Seldon Core, including authentication basics."
summary: "Deploy a first machine learning model on Kubernetes using Seldon Core, including authentication basics."
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

## Setup environments

[Link here!](/seldon-core-serving/inference-graph/README.md)

## Training a model

```sh
$ python3 train.py
*
optimization finished, #iter = 8
obj = -4.874554, rho = -0.126317
nSV = 11, nBSV = 8
*
optimization finished, #iter = 17
obj = -2.130718, rho = 0.064464
nSV = 8, nBSV = 2
*
optimization finished, #iter = 37
obj = -34.058808, rho = 0.107043
nSV = 47, nBSV = 45
Total nSV = 60
```

## Write a class wrapper that exposes the logic of your model

```python
import pickle
from sklearn import svm

class IrisClassifier:
    def __init__(self):
        self._model: svm.SVC = pickle.load(open("model.pkl", "rb"))

    def predict(self, X, features_names=None, meta=None):
        output = self._model.predict(X)
        return output
```

## Build image

```sh
$ docker build -t hoangph3/sklearn_iris_classifier:v0.0.1 .
```

## Deploy the model

1. Create a namespace to run your model in:

```sh
$ kubectl create namespace seldon-model
namespace/seldon created
```

1. Create the manifest file `deploy-model.yaml`

```yaml
apiVersion: machinelearning.seldon.io/v1alpha2
kind: SeldonDeployment
metadata:
  name: iris-model
  namespace: seldon-model
spec:
  name: iris
  predictors:
  - componentSpecs:
    - spec:
        containers:
        - name: classifier
          image: hoangph3/sklearn_iris_classifier:v0.0.1
    graph:
      name: classifier
    name: default
    replicas: 1
```

1. Deploy it to our Seldon Core Kubernetes Cluster:

```sh
$ kubectl apply -f deploy-model.yaml
seldondeployment.machinelearning.seldon.io/iris-model created

$ kubectl get pods -n seldon-model
NAME                                               READY   STATUS    RESTARTS   AGE
iris-model-default-0-classifier-64d68cbd8d-tj2r5   3/3     Running   0          38s
```

## Testing

```sh
$ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

$ echo $INGRESS_HOST
10.106.254.233

$ curl -X POST -d @payload.json -H 'Content-Type: application/json' \
    http://$INGRESS_HOST/seldon/seldon-model/iris-model/api/v1.0/predictions | json_pp

{
   "data" : {
      "ndarray" : [
         2
      ],
      "names" : []
   },
   "meta" : {
      "requestPath" : {
         "classifier" : "hoangph3/sklearn_iris_classifier:v0.0.1"
      }
   }
}
```

## Authentication and Authorization

## Authentication and Authorization for Seldon Core Requests

This is an example of setting up auth for seldon core model deployments in an istio enabled kubernetes cluster. Here we discuss the following topics:

- Authentication at Seldon Deployment Component Level
- Authorization based on user id token claims
- Authentication at the Ingress Level

You can get full source code here: [first-model](https://github.com/hoangph3/mlops-labs/tree/main/seldon-core-serving/first-model), [auth](https://github.com/hoangph3/mlops-labs/tree/main/seldon-core-serving/auth).
