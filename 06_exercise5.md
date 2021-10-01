---
layout: page
permalink: /liberty-workshop/exercise05
---
__Exercise 5__

With our application now deployed into Dev, let's have a look at an alternative method of deployment for production. We will look to apply the ideals of GitOps, where the source of truth for our environment is stored in Git, and the environment continuously converges on this desired state.

To do so, we will exploit the declarative nature of Kubernetes using a tool called OpenShift GitOps, which is based on the upstream project ArgoCD.

In this exercise we will:
1. Upload our application's Kube manifests into git
1. Set up an individual ArgoCD instance and project
1. Demonstrate how changes in Git are synchronised in our cluster

#### Step 1
We will kick off by creating a production namespace for ourselves:
```bash
export NAMESPACE=${USER}-petclinic-prod

oc new-project $NAMESPACE
```

#### Step 2
Now let's define our application in Git.

Start by cloning the empty project:
```bash
cd ~

git clone ${GITEA_URL}/${USER}/petclinic-prod

cd petclinic-prod
```

Create your Petclinic application manifest, which will include an Open Liberty application with a persistent volume:
```bash
echo "apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: petclinic-serviceability
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ocs-storagecluster-cephfs
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi" > petclinic-serviceability-pvc.yml

echo "apiVersion: openliberty.io/v1beta1
kind: OpenLibertyApplication
metadata:
  name: liberty-petclinic
spec:
  expose: true
  applicationImage: ${REGISTRY_URL}/${USER}/petclinic:v1.0.0
  serviceability:
    size: 1G
    volumeClaimName: petclinic-serviceability
  replicas: 1" > petclinic-olapp.yml
```

Now, commit and push your changes into git:
```bash
git add petclinic-serviceability-pvc.yml petclinic-olapp.yml

git commit -m "Add Petclinic application definition"

git push
```

#### Step 3
Now that we have our application's source of truth defined in Git, it is time to set up ArgoCD to work its magic. We will be using the ArgoCD operator to deploy and manage our ArgoCD instances and projects.

We are going to take a couple of shortcuts here with a two main goals in mind:
* Simplify the deployment and configuration
* Give everyone an individual ArgoCD instance, allowing the more adventurous to experiment without impacting others

In this step, we will deploy an individual ArgoCD instance in our production namespace, alongside our application. In a typical production environment, there will likely be a single "master" ArgoCD server in a protected administrative namespace.

First, create your ArgoCD instance:
```bash
echo "apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  labels:
    app: argocd
  name: argocd
spec:
  server:
    route:
      enabled: true" | oc apply -n $NAMESPACE -f -
```

Once complete, you should have a functional ArgoCD instance. Grab the route (you know how right?!), open it in a browser, log in and have a poke around.

You could define your Petclinic project and application via this web console, but we like things stored as code, and using the ArgoCD Operator we can easily do exactly that.

Define your Petclinic project, which is essentially a "namespace" in ArgoCD. The project definition includes a list of "allowed" git sources and destination clusters/namespaces. Pay attention to those environment variables!
```bash
echo "apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: petclinic
spec:
  destinations:
  - name: in-cluster
    namespace: ${USER}-petclinic-prod
    server: https://kubernetes.default.svc
  sourceRepos:
  - ${GITEA_URL}/${USER}/petclinic-prod" | oc apply -n $NAMESPACE -f -
```

Now define your ArgoCD Petclinic application (an "application" in this sense is a construct within ArgoCD). This is where we will specify the exact source git repo and destination cluster/namespace:
```bash
echo "apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: petclinic
spec:
  destination:
    name: in-cluster
    namespace: ${USER}-petclinic-prod
  project: petclinic
  source:
    path: ./
    repoURL: ${GITEA_URL}/${USER}/petclinic-prod
  syncPolicy:
    automated:
      selfHeal: true" | oc apply -n $NAMESPACE -f -
```

Jump back into the ArgoCD web console and watch your changes take place.

Once your Petclinic application is synchronised, have a quick explore of the resources (pods, services, routes, volumes, etc) in the prod namespace. It should all feel very familiar since you've deployed this before. The major difference now is that you defined your Open Liberty application entirely in Git and let the cluster take care of the rest. We'll further explore what this means in practice in the next step.

#### Step 4
Now it's time to make a change to our source of truth. This would likely be a new feature or patched version of our Petclinic application, but let's save that for the next exercise.

Here, will make a very simple change and increase the number of pods in our application.

Edit your Open Liberty application and increase the number of replicas to 2:
```bash
cd ~/petclinic-prod

sed -i 's/replicas: 1/replicas: 2/' petclinic-olapp.yml

git add petclinic-olapp.yml

git commit -m "Increase replica count to 2"

git push
```

Now head back over to the ArgoCD console. In the previous steps, we set up ArgoCD to sync automatically and every 30 seconds. Give it a moment and you will see your application start to synchronise, converging on the desired state defined in Git.

Once complete, confirm that your Liberty application now has two running pods.

#### Stretch Goal
<TODO>

[Previous Exercise](exercise04) / [Next Exercise](exercise06)
