---
layout: page
permalink: /liberty-workshop/exercise04
---
__Exercise 4__

In this exercise we will spend some time exploring, building, and customising container images.

Once again we'll use the Petclinic application, though of course with another subtle twist. This time, we will take a prebuilt application (a fat JAR with a Tomcat server built in), optimise it for running on Liberty, and bake it into an image ready for deployment. While we do this, we'll also apply some custom security hardening to our builder image.

In this exercise we will:
1. Apply patches and custom security fixes to a container image
1. Take an existing Java application and optimise it for Liberty
1. Deploy the application into a new development namespace

#### Step 1
First up, let's get some prep tasks out of the way.

Clone the repository containing the container image build files (aka Dockerfiles):
```bash
git clone https://github.com/mattparko/dockerfiles ~/dockerfiles
```

Pull the build image from Dockerhub:
```bash
podman pull docker.io/openliberty/open-liberty:springBoot2-ubi-min
```

Login in to our Nexus container image registry:
```bash
podman login -u $USER -p openshift $REGISTRY_URL
```

#### Step 2
Now it's time to build our new image and push it to our Nexus registry.

First, take a look at the container file we will use for the build. Notice we applying the latest patches, as well as executing some security hardening scripts. These scripts are taken from the US DoD Iron Bank project.
```bash
cd ~/dockerfiles

cat Dockerfile.custom-spring-ubi
```

Now execute the image build:
```bash
podman build -t ${REGISTRY_URL}/${USER}/openliberty:spring-java8 -f Dockerfile.custom-spring-ubi .

podman images
```

Finally, push your built image into the Nexus image registry. We'll use this image in a later step:
```bash
podman push ${REGISTRY_URL}/${USER}/openliberty:spring-java8
```

While the Nexus Registry does not perform container scans, the Red Hat Quay image registry does. I pushed both the before and after images into my personal Quay repository to illustrate the difference in scan results. Head to https://quay.io/repository/mparkins/openliberty?tab=tags to check it out.

#### Step 3
In this step, we will use the secure image we just created to optimise an existing Spring application for running on Liberty. The resulting optimised application will be packaged into a container using the same image and pushed to our container image registry.

The container build file has already been provided (`~/dockerfiles/Dockerfile.spring-tomcat-to-liberty`). Before moving on, take a second to familiarise yourself with the contents of this build file. Notice the following:
- This is a multi-stage build, meaning we will run steps in a container and then use the output to build another container. A multi-stage build allows us to cut down on the size and number of layers in our final container image
- There is a variable named `IMAGE` with a defined default value. We will override this variable during the build
- In the first stage we are running a utility named `springBootUtility` to create our thin JAR
- The second stage of the build places the thin app and extracted libraries into the relevant Open Liberty directories, ensuring permissions are correct

The existing application (fat JAR) has been uploaded already to `${NEXUS_URL}/repository/maven-releases/com/spring-petclinic/v1.0.0/spring-petclinic-v1.0.0.jar`.

Let's go ahead and kick off the container build:
```bash
cd ~/dockerfiles

podman build -t ${REGISTRY_URL}/${USER}/petclinic:v1.0.0 -f Dockerfile.spring-tomcat-to-liberty .
```

Once complete, we will again push the final image into our registry:
```bash
podman push ${REGISTRY_URL}/${USER}/petclinic:v1.0.0
```

#### Step 4
Action time! Let's check out our newly optimised Liberty application.

We'll let the Liberty operator do the hard work again:
```bash
export NAMESPACE=${USER}-petclinic-dev

oc project $NAMESPACE

echo "apiVersion: openliberty.io/v1beta1
kind: OpenLibertyApplication
metadata:
  name: liberty-petclinic
spec:
  expose: true
  applicationImage: ${REGISTRY_URL}/${USER}/petclinic:v1.0.0
  replicas: 1" | oc apply -n $NAMESPACE -f -
```

#### Stretch Goal
You should now have two Liberty applications running in the same namespace. Set up a new route that splits incoming traffic between the two applications, effectively simulating a blue-green deployment. Name the route whatever you would like and weight the traffic to each application version however you see fit.

__Hint:__ use the OpenShift web console for this one, and make sure you are in the "Administrator" view (i.e. not the "Developer" view).

[Previous Exercise](exercise03) / [Next Exercise](exercise05)
