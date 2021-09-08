---
layout: page
permalink: /liberty-workshop/exercise01
---
__Exercise 1__

By now you should be happily logged in to both the OpenShift Web Console and via the command line tool (`oc`).

Let's get started with deploying our first application!

In this exercise we will:
1. Create your first project
1. Create persistent storage for your application
1. Deploy an application using the Open Liberty Operator

#### Step 1
Create an OpenShift project (namespace) and add it to your environment variables.
```bash
oc new-project $USER

export NAMESPACE=$USER
```

#### Step 2
Create a persistent ReadWriteMany (RWX) storage claim, provided by OpenShift Container Storage. This storage will later be utilised by the Open Liberty trace and dump features.

This method applies a Kubernetes manifest definition to the cluster by "piping" the definition to the `oc apply` command. You can also store the definition as a YAML file and apply with `oc apply -f file.yml`. It's also possible to create most objects via the Web Console, but of course there are many benefits to manifest files (and the command line) when it comes to automation.
```bash
echo "apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-app-serviceability
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi" | oc apply -n $NAMESPACE -f -
```

#### Step 3
Deploy an existing sample Open Liberty application using the Open Liberty Operator.
```bash
echo "apiVersion: openliberty.io/v1beta1
kind: OpenLibertyApplication
metadata:
  name: demo-app
  labels:
    app: frontend
spec:
  expose: true
  applicationImage: >-
    registry.connect.redhat.com/ibm/open-liberty-samples@sha256:8ea2d3405ff2829d93c5dda4dab5d695ea8ead34e804aaf6e39ea84f53a15ee4
  serviceability:
    size: 1G
    volumeClaimName: demo-app-serviceability
  replicas: 1" | oc apply -n $NAMESPACE -f -
```
Take note of the various components like the application name, the application image, and the serviceability section, which includes the persistent storage we previously created.

#### Step 4
Explore your deployed Open Liberty application!

Take a look at the managed Open Liberty application object.
```bash
oc get openlibertyapplications
oc get openlibertyapplication demo-app -o yaml
```

Take a look at the OpenShift resources deployed via the operator:
```bash
oc get deployments
oc get services
oc get replicasets
oc get secrets
oc get routes
```
Follow the URL found in the route to see your deployed sample application.

#### Step 5
Try to delete some resources. Wait and watch to see what happens:
```bash
oc delete deployment demo-app
oc delete route demo-app
```
Why are they not being deleted properly? (Hint: what is looking after your Liberty application?)

#### Stretch Goal
Try to scale up the number of deployed pods in your application. You can do this via the Web Console, or with a command like:
```bash
oc scale deployment demo-app --replicas=2
```
Again, why isn't this working properly?

How could you scale up your Open Liberty application? How could you do it via the Web Console? What about via the command line?


[Previous Exercise](setup) / [Next Exercise](exercise02)
