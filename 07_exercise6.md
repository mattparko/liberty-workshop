---
layout: page
permalink: /liberty-workshop/exercise06
---
__Exercise 6 - CI pipelines with Tekton__

With our Petclinic application optimised for Liberty and deployed, let's take a look at some automated approaches to help reduce repetition and toil. We're going to create a CI/CD workflow using a feature called OpenShift Pipelines, which itself is based on the upstream open source project - Tekton.

We will use this pipeline to automate the Tomcat to Open Liberty migration that was performed in [Exercise 4](exercise04). This would allow us to easily repeat the process for many applications. The pipeline will deploy our application into a test namespace, run some stubbed test cases, and then on success will update our production application via OpenShift GitOps (ArgoCD).

In this exercise we will:
1. Setup our namespaces and policies
1. Create a Tekton pipeline
1. Build and deploy our application using the pipeline

#### Step 1 - Preparation
Before we get stuck into OpenShift Pipelines, let's set up a namespace we can run our pipelines in. This doesn't necessarily need to be a separate namespace, but it can make things neater and more organised. And besides, we are here to practice right?!

First of all, create the namespace:
```bash
export NAMESPACE=${USER}-pipelines

oc new-project $NAMESPACE
```

Tekton tasks run as individual containers within the OpenShift platform. This enables excellent scale and performance, but it also results in zero resource usage when not executing a pipeline. Since our tasks are run in different containers though, we will need a shared workspace to provide persistent storage between our various Tekton tasks:

Create a new persistent volume claim (PVC):
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

Our Tekton pipeline will build and push images into our private container image repository. This means it needs to know our credentials. There are a number of more secure ways to approach this, but for simplicity we will provide our personal Nexus credentials. These credentials were supplied to podman previously, which helpfully caches them in the exact format needed for the OpenShift secret.

Go ahead and create the registry secret:
```bash
echo "apiVersion: v1
kind: Secret
metadata:
  name: nexus-pull-secret
data:
  .dockerconfigjson: $(cat ${XDG_RUNTIME_DIR}/containers/auth.json | base64 -w0)
type: kubernetes.io/dockerconfigjson" | oc apply -n $NAMESPACE -f -
```

Now create a link between the registry secret and the relevant service accounts:
```bash
oc secrets link builder nexus-pull-secret --for=pull,mount -n $NAMESPACE

oc secrets link pipeline nexus-pull-secret --for=pull,mount -n $NAMESPACE
```

Lastly, as we are going to use Tekton to update our production Git repository directly, we also need to include our git credentials. This example uses HTTP basic auth, but this is **strongly** discouraged. It is far better to create a deploy key and use ssh key authentication.

Create the secret, but be careful with the substitution of the Gitea URL. Notice that in the `.git-credentials` section the URL is split as the username and password are inserted:
```bash
export GITEA_URL=<Gitea URL from Etherpad>

export PASS=<Gitea password>

export GITEA_HOSTNAME=$(echo $GITEA_URL | sed -e 's%https://%%')

echo "kind: Secret
apiVersion: v1
metadata:
  name: git-secret
type: Opaque
stringData:
  .gitconfig: |
    [credential \"https://${GITEA_HOSTNAME}\"]
      helper = store
  .git-credentials: |
    https://${USER}:${PASS}@${GITEA_HOSTNAME}" | oc apply -n $NAMESPACE -f -
```

#### Step 2 - Create a pipeline
Next we are going to create our Tekton pipeline. A pipeline is made up of tasks, with both the pipeline and the tasks defined in YAML (of course). You can learn more about Tekton from the [official docs](https://docs.openshift.com/container-platform/4.8/cicd/pipelines/understanding-openshift-pipelines.html). There is also a handy comparison between Tekton and Jenkins concepts for those more familiar with Jenkins [here](https://docs.openshift.com/container-platform/4.8/cicd/jenkins-tekton/migrating-from-jenkins-to-tekton.html).

The Tekton YAML manifests have been created already and are stored in GitHub. Clone the repository locally:
```bash
cd ~

git clone https://github.com/mattparko/liberty-workshop-extras

cd liberty-workshop-extras/petclinic-pipelines
```

Feel free to explore the manifests.

Now create the Tekton tasks:
```bash
for MANIFEST in $(ls task-*); do
  oc apply -f $MANIFEST -n $NAMESPACE
done
```

Once complete, create the Tekton pipeline:
```bash
oc apply -f pipeline-spring-tomcat-to-liberty.yaml -n $NAMESPACE
```

#### Step 3 - Create a test namespace
Now we are going to set up our test namespace in OpenShift. This is where our Tekton pipeline will stage our application for testing.

Create the namespace:
```bash
export NAMESPACE=${USER}-petclinic-test

oc new-project $NAMESPACE
```

Remember though that namespaces are separate projects, with separate service accounts, roles, permissions, etc. So we will need to allow our Tekton service account (from the `$USER-pipelines` namespace) to deploy into our test namespace. To do so we will create a role binding policy:
```bash
oc policy add-role-to-user edit system:serviceaccount:${USER}-pipelines:pipeline --rolebinding-name=pipeline-edit -n $NAMESPACE
```

#### Step 4 - Define a pipeline run
We are almost ready to execute our pipeline.

OpenShift Pipelines expose a number of variables allowing you to reuse the pipeline with different inputs. When you execute a pipeline, an object known as a PipelineRun is created, which contains all of our populated input variables. Because the PipelineRun is just another Kubernetes object, this allows us to define our own PipelineRun using a YAML manifest, which is then used to execute the pipeline.

Other approaches to executing a pipeline include using the web console and manually filling out the variables in a form, or using the Tekton CLI tool and including the input values on the command line (see `tkn pipeline start -h`).

Below, we will take the approach of defining a PipelineRun using a YAML manifest. You are welcome to try other approaches.

Switch back to your pipelines namespace:
```bash
export NAMESPACE=${USER}-pipelines

oc project $NAMESPACE
```

Create the pipeline run YAML file. Pay attention to environment variables. They will be substituted if set up correctly, otherwise you will need to manually update your file.
```bash
echo "apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  annotations:
    description: |
      Take a pre-built Spring fat jar and optimise for IBM Liberty
  labels:
    tekton.dev/pipeline: spring-tomcat-to-liberty
  generateName: spring-tomcat-to-liberty-
  namespace: $NAMESPACE
spec:
  params:
  - name: APP_NAME
    value: petclinic
  - name: IMAGE_NAME
    value: ${REGISTRY_URL}/${USER}/petclinic
  - name: SOURCE_JAR
    value: ${NEXUS_URL}/repository/maven-releases/com/spring-petclinic/v1.0.0/spring-petclinic-v1.0.0.jar
  - name: BUILDER_IMAGE_REPO
    value: https://github.com/mattparko/dockerfiles
  - name: BUILDER_IMAGE_REPO_REVISION
    value: main
  - name: BUILDER_FILE
    value: ./Dockerfile.spring-tomcat-to-liberty
  - name: BUILDER_BASE_IMAGE
    value: ${REGISTRY_URL}/${USER}/openliberty:spring-java8
  - name: DEPLOY_TO_TEST
    value: 'true'
  - name: NAMESPACE_TEST
    value: ${USER}-petclinic-test
  - name: DEPLOY_TO_PROD
    value: 'true'
  - name: PROD_GITOPS_REPO
    value: ${GITEA_URL}/${USER}/petclinic-prod.git
  - name: PROD_LIBERTY_MANIFEST
    value: ./petclinic-olapp.yml
  - name: PROD_GITOPS_USER
    value: $USER
  - name: PROD_GITOPS_EMAIL
    value: ${USER}@example.opentlc.com
  pipelineRef:
    name: spring-tomcat-to-liberty
  serviceAccountName: pipeline
  timeout: 1h0m0s
  workspaces:
  - name: app-source
    persistentVolumeClaim:
      claimName: workspace-pvc
  - name: git-credentials
    secret:
      secretName: git-secret
  - name: git-cli-working-dir
    emptydir: {}" > ~/my-pipeline-run.yml
```

#### Step 5 - Execute the pipeline
Now go ahead and execute the pipeline!
```bash
oc create -f ~/my-pipeline-run.yml
```

Jump into the web console to watch the progress of your pipeline run. Make sure you are on the correct project (`<USER>-pipelines`). You should be in the Developer view (i.e. not the Administrator view) and you will see the `Pipelines` option in the left pane. Select your pipeline and then select the `Pipeline Runs` tab. Select the latest pipeline - you should get a live view of your pipeline steps pass/fail status, as well as the ability to view logs from each of the tasks.

Keep an eye on your test and prod namespaces for updates too.

Once complete, confirm that ArgoCD has deployed your shiny new container image into prod. To do this, switch to the prod namespace, list the Open Liberty applications, and check the image name and tag. You should see an image with a very recent timestamp as a tag. This timestamp was generated by the Tekton pipeline and used as an image tag.
```bash
oc project ${USER}-petclinic-prod

oc get openlibertyapplications
```

Feel free to run the pipeline again - you will see an updated image with a newer timestamp once complete!

#### Stretch Goal
Go have some fun!

Experiment, try out new ideas, try to break something (preferably not your colleagues namespaces though). Now is the time to let your curiosity run wild.

[Previous Exercise](exercise05)
