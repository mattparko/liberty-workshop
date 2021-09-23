---
layout: page
permalink: /liberty-workshop/exercise06
---
__Exercise 6__

OpenShift Pipelines
- Pipeline deploy to test and then prod

With our Petclinic application optimised for Liberty and manually deployed, let's take a look at some automated approaches to help reduce repetition and toil. We're going to create a CI/CD workflow using a feature called OpenShift Pipelines, which itself is based on the upstream open source project - Tekton.

We will use this pipeline to automate the Tomcat to Open Liberty migration that was performed in [Exercise 4](exercise04). This would allow us to easily repeat the process for many applications. The pipeline will deploy our application into a test namespace, run some stubbed test cases, and then on success will update our production application via OpenShift GitOps (ArgoCD).

In this exercise we will:
1. Setup our namespaces and policies
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

#### Step 2
Next we are going to create our Tekton pipeline. A pipeline is made up of tasks, with both the pipeline and the tasks defined in YAML (of course). You can learn more about Tekton from the [official docs](https://docs.openshift.com/container-platform/4.8/cicd/pipelines/understanding-openshift-pipelines.html). There is also a handy comparison between Tekton and Jenkins concepts for those more familiar with Jenkins [here](https://docs.openshift.com/container-platform/4.8/cicd/jenkins-tekton/migrating-from-jenkins-to-tekton.html).

The Tekton YAML manifests have been created already and are stored in GitHub. Clone the repository locally:
```bash
cd ~

git clone <TODO>

cd <TODO>
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

#### Step X
Now we are going to set up our destination namespace. This is where our Tekton pipeline will deploy our application.

Create the namespace:
```bash
export NAMESPACE=${USER}-petclinic-test

oc new-project $NAMESPACE
```

Remember though that namespaces are separate project, with separate roles, permissions, etc. So we will need to allow our pipeline service account to deploy into our test namespace. To do so we will create a role binding policy:
```bash
oc policy add-role-to-user edit system:serviceaccount:${USER}-pipelines:pipeline --rolebinding-name=pipeline-edit -n $NAMESPACE
```

#### Step X
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
    value: "${NEXUS_URL}/repository/maven-releases/com/spring-petclinic/v1.0.0/spring-petclinic-v1.0.0.jar" 
  - name: BUILDER_IMAGE_REPO
    value: https://github.com/mattparko/dockerfiles
  - name: BUILDER_IMAGE_REPO_REVISION
    value: main
  - name: BUILDER_FILE
    value: ./Dockerfile.spring-tomcat-to-liberty
  - name: DEPLOY_TO_TEST
    value: "true"
  - name: NAMESPACE_TEST
    value: ${USER}-petclinic-test
  - name: DEPLOY_TO_PROD
    value: "true"
  - name: PROD_GITOPS_REPO
    value: git@${<TODO>}/petclinic-argocd.git
  - name: PROD_LIBERTY_MANIFEST
    value: ./prod/petclinic-olapp.yaml
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
  - name: ssh-creds
    secret:
      secretName: github-public-demo-deploy-key
  - name: git-cli-working-dir
    emptydir: {}" > ~/my-pipeline-run.yml
```

#### Step X
Now go ahead and execute the pipeline!
```bash
oc create -f ~/my-pipeline-run.yml
```

Jump into the web console to watch the progress of your pipeline run. Make sure you are on the correct project (`<USER>-pipelines`).

#### Stretch Goal
Go have some fun!

Experiment, try out new ideas, try to break something (preferably not your colleagues namespaces though). Now is the time to let your curiosity run wild.

[Previous Exercise](exercise05)
