---
layout: page
permalink: /liberty-workshop/exercise01
---
__Exercise 1 - Deploy an Open Liberty application__

By now you should be happily logged in to both the OpenShift Web Console and via the command line tool (`oc`).

Let's get started with deploying our first application!

In this exercise we will:
1. Create your first project
1. Create persistent storage for your application
1. Deploy an application using the Open Liberty Operator

#### Step 1 - Create a project
Create an OpenShift project (namespace) and add it to your environment variables.
```bash
oc new-project $USER

export NAMESPACE=$USER
```

If the project created successfully, you should see output similar to this:
```text
Now using project "user1" on server "https://api.openshift.example.com:6443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app rails-postgresql-example

to build a new example application in Ruby. Or use kubectl to deploy a simple Kubernetes application:

    kubectl create deployment hello-node --image=k8s.gcr.io/serve_hostname
```

You can always check which project you are currently using by running the `oc project` command:
```bash
$ oc project
Using project "user1" on server "https://api.openshift.example.com:6443".
```

#### Step 2 - Create persistent storage
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

You should see output similar to:
```text
persistentvolumeclaim/demo-app-serviceability created
```

You can view the status of all persistent volume claims in your namespace with the command `oc get pvc`, or you can list your specific PVC resource with `oc get pvc demo-app-serviceability`.

To see all information about the resouce, use the command `oc get pvc demo-app-serviceability -o yaml`. This displays all known information about the OpenShift resource in YAML format, including its definition, metadata, and current status. This sort of detailed output is available for all OpenShift resources that you have authority to view.

You can also view the volume (including YAML!) in the web console. Make sure you select the correct project at the top of the page and then, in the left pane, select `Storage` and `PersistentVolumeClaims`. Select your PVC and check out the details available.

#### Step 3 - Deploy an application
Now let's deploy a sample Open Liberty application using the Open Liberty Operator. This deployment will use an existing container image with a sample Liberty application pre-baked into it. Notice the various components like the application name, the application image, and the serviceability section, which includes the persistent storage we previously created.
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

You should see output similar to:
```text
openlibertyapplication.openliberty.io/demo-app created
```

#### Step 4 - View application resources
Explore your deployed Open Liberty application!

Head over to the web console and look at some of the resources. You can switch between both Administrator and Developer views to get an idea of what (and how) information is presented in each view.

Jump back into the command line on the bastion host and take a look at the managed Open Liberty application object.
```bash
oc get openlibertyapplications
oc get openlibertyapplication demo-app -o yaml
```

_Note:_ There are "short names" for many of OpenShift's resources and an Open Liberty application is no exception. You can also use `oc get olapp` and `oc get olapps`. Also worthy of note is that plurals make no difference - they just help to support a more human-friendly approach.

Take a look at the OpenShift resources deployed via the operator:
```bash
oc get deployments
oc get services
oc get replicasets
oc get secrets
oc get routes
```

Follow the URL found in the route to see your deployed sample application. If you see an error page, just be patient - your application is likely still being deployed. Feel free to view the logs in the Liberty container:
```bash
oc get pods

oc logs <demo-app pod name>
```

#### Step 5 - Delete some components
Try to delete some resources. Wait and watch the topology view in the web console to see what happens:
```bash
oc delete deployment demo-app
oc delete route demo-app
```
Why are they not being deleted properly? Hint: what is looking after your Liberty application and all of its components?

We are demonstrating here that the Open Liberty operator continues to maintain control of your application's state and resources.

#### Stretch Goal
Try to scale up the number of deployed pods in your application. You can do this via the Web Console, or with a command like:
```bash
oc scale deployment demo-app --replicas=2
```
Again, why isn't this working properly?

How could you scale up your Open Liberty application? How could you do it via the Web Console? What about via the command line?


[Previous Exercise](setup) / [Next Exercise](exercise02)
