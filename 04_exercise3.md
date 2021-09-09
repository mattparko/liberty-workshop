---
layout: page
permalink: /liberty-workshop/exercise03
---
__Exercise 3__

Now that we have explored the Open Liberty operator, let's take a look at some of features of the OpenShift platform - especially around managing application deployments.

We'll again be using the Petclinic sample application over the next few exercises but this time, instead of using an existing container image with the application baked in, we'll look at what's required to build that image ourselves. This is much more similar to what will be required in your organisation, where you may be tasked to take some source code or an application binary and package it inside a container.

In this exercise we will:
1. Explore a Source-to-Image (S2I) build
1. 

#### Step 1
Clone the Spring Petclinic project and push it into your Gitea repository:
```bash
export GITEA_URL=<Gitea URL from Etherpad>

git clone https://github.com/spring-projects/spring-petclinic

git remote set-url origin https://${GITEA_URL}/${USER}/spring-petclinic

git push
```

Let's also create a git branch specifically for development:
```bash
git checkout -b development

git push --set-upstream origin development
```

#### Step 2
Now, create a new project in OpenShift. This is where we will build and deploy our new Petclinic application:
```bash
export NAMESPACE=${USER}-petclinic-dev

oc new-project ${NAMESPACE}
```

#### Step 3
This is where things start to get interesting!

In this next step we are going to ask OpenShift to create an application for us, however best it sees fit. You can provide as much or as little guidance here and the platform will decide on the best build strategy. Probably not the way you would deploy an application into production, but an excellent way to get developers started as quickly as possible.
```bash
oc new-app --name=petclinic registry.access.redhat.com/ubi8/openjdk-11~https://${GITEA_URL}/${USER}/spring-petclinic#development
```

Let's explain what is happening here:
1. `oc new-app` creates all of the components you need for an OpenShift application, whether from source code, binaries, templates, or images.
1. `--name=petclinic` gives our application a name (this is optional - OpenShift will make up a name based on other info, like the git repo name)
1. `registry.access.redhat.com/ubi8/openjdk-11` is the image we want to use to build our application (also optional - OpenShift will pick a builder image based on the source code type)
1. `~https://${GITEA_URL}/${USER}/spring-petclinic#development` is our source code repository - specifically the development branch

In essence, OpenShift is pulling the builder image, copying the source code into it, building the application, and then creating a new container image with our application bundled inside. This is known as a Source-to-Image, or S2I, build. 

#### Step 4
The build in the previous step will take a few minutes to complete, so let's follow along with it.

Check out the resources (eg. deployment, service, pods) that have been created:
```bash
oc get all
```

Have a look at the event logs:
```bash
oc get events
```

Following along with the application build currently taking place in the builder image:
```bash
oc get pods

# Use ctrl-c to stop tracking logs at any time
oc logs -f petclinic-1-build
```

#### Step 5
Once your build is complete, create a route for your application and check out the result in a web browser:
```bash
oc expose service petclinic

oc get routes
```

#### Step 6
You have exposed your application using an insecure HTTP endpoint. How could you utilise the cluster's TLS certificate instead?

Hints:
* `oc create route --help`
* `oc create route edge --help`

[Previous Exercise](exercise02) / [Next Exercise](exercise04)
