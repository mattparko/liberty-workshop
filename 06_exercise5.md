---
layout: page
permalink: /liberty-workshop/exercise05
---
__Exercise 5__

OpenShift Pipelines

- Pipeline deploy to test?

With our Petclinic application optimised for Liberty and manually deployed, let's take a look at some automated approaches to help reduce repetition and toil. We're going to create a CI/CD workflow using a feature called OpenShift Pipelines, which itself is based on the upstream open source project - Tekton.

In this exercise we will:
1. Define some custom Tekton tasks
1. Create a Tekton pipeline
1. Build and deploy our application using the pipeline

#### Step 1
Before we get stuck into OpenShift Pipelines, let's set up a namespace we can run our pipelines in. This doesn't necessarily need to be a separate namespace, but it can make things neater and more organised. And besides, we are here to practice right?!

First of all, create the namespace:
```bash
export NAMESPACE=${USER}-pipelines

oc new-project $NAMESPACE
```

Create a new physical volume claim (PVC). This PVC will be used as a workspace to provide persistent storage between our various Tekton tasks:
```bash
echo "apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: workspace-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi" | oc apply -n $NAMESPACE -f -
```



#### Step X
Now we are going to set up our destination namespace. This is where our Tekton pipeline will deploy our application.

Create the namespace:
```bash
export NAMESPACE=${USER}-petclinic-test

oc new-project $NAMESPACE
```

Remember though that namespaces are separate project, with separate roles, permissions, etc. So we will also need to allow

[Previous Exercise](exercise04) / [Next Exercise](exercise06)
