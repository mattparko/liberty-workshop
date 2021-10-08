---
layout: page
permalink: /liberty-workshop/exercise05
---
__Exercise 5 - GitOps with ArgoCD__

With our application now deployed into Dev, let's have a look at an alternative method of deployment for production. We will look to apply the ideals of GitOps, where the source of truth for our environment is stored in Git, and the environment continuously converges on this desired state.

To do so, we will exploit the declarative nature of Kubernetes using a tool called OpenShift GitOps, which is based on the upstream project ArgoCD.

In this exercise we will:
1. Upload our application's Kube manifests into git
1. Set up an individual ArgoCD instance and project
1. Demonstrate how changes in Git are synchronised in our cluster

#### Step 1 - Create a production namespace
We will kick off by creating a production namespace for ourselves:
```bash
export NAMESPACE=${USER}-petclinic-prod

oc new-project $NAMESPACE
```

#### Step 2 - Define our Liberty application in Git
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

Notice there is nothing particularly new about these resources - they are very similar to the persistent volume and Open Liberty application created in earlier exercises. The difference now is that we not created the resources in our cluster - we have simply saved their definitions into a text file.

Now, commit and push your changes into git:
```bash
git add petclinic-serviceability-pvc.yml petclinic-olapp.yml

git commit -m "Add Petclinic application definition"

git push
```

#### Step 3 - Setup ArgoCD
Now that we have our application's source of truth defined in Git, it is time to set up ArgoCD to work its magic. We will be using the ArgoCD operator to deploy and manage our ArgoCD instances and projects.

We are going to take a couple of shortcuts here with two main goals in mind:
* Simplify the deployment and configuration
* Give everyone an individual ArgoCD instance, allowing the more adventurous to experiment without impacting others

In this step, we will deploy an individual ArgoCD instance in our production namespace, alongside our application. In a typical production environment, there will likely be a single "master" ArgoCD server in a protected administrative namespace - potentially even a separate management cluster.

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

Once complete, you should have a functional ArgoCD instance. Grab the route (you know how right?!) and open it up in a browser. To log in we need the admin password. This is stored in a Kubernetes secret. Retrieve it with the following command:
```bash
oc get secrets argocd-cluster -o jsonpath='{.data.admin\.password}' | base64 -d && echo
```

Use this password, along with the username `admin` to log in.

You could define your Petclinic project and application via ArgoCD's web console, but we like things stored as code and using the ArgoCD Operator we can easily do exactly that.

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

Once your Petclinic application is synchronised, have a quick explore of the OpenShift resources (pods, services, routes, volumes, etc) in your prod namespace. It should all feel very familiar since you've deployed this before. The major difference now is that you defined your Open Liberty application entirely in Git and let the cluster take care of the rest. We'll further explore what this means in practice in the next step.

#### Step 4 - Make a change to the application
Now it's time to make a change to our source of truth. This would likely be a new feature or patched version of our Petclinic application, but let's save that for the next exercise.

Here, we will make a very simple change and increase the number of pods in our application.

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

What you just achieved was a change in production via nothing but a Git commit and push. Combine this with a solid CI pipeline and both automated and manual approval gates, suddenly your entire release and lifecycle management processes just became a whole lot easier for all involved.

#### Stretch Goal
Consider the next exercise in its entirety a stretch goal - it's a doozy!

Exercise 6 ties together everything we've done so far using a CI/CD pipeline. Good luck!

[Previous Exercise](exercise04) / [Next Exercise](exercise06)
