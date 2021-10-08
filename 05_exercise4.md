---
layout: page
permalink: /liberty-workshop/exercise04
---
__Exercise 4 - Customising base images__

In this exercise we will spend some time exploring, building, and customising container images.

Once again we'll use the Petclinic application, though of course with another subtle twist. This time, we will take a prebuilt application (a fat JAR with a Tomcat server built in), optimise it for running on Liberty, and bake it into an image ready for deployment. While we do this, we'll also apply some custom security hardening to our builder image.

In this exercise we will:
1. Apply patches and custom security fixes to a container image
1. Take an existing Java application and optimise it for Liberty
1. Deploy the application into a new development namespace

#### Step 1 - Preparation
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
export REGISTRY_URL=<Registry URL from Etherpad>

podman login -u $USER -p openshift $REGISTRY_URL
```

_Note_: We are using Dockerhub as a read-only, unauthenticated, and anonymous user. We logged in to our private container image registry so that we can push images to it (i.e. upload content).

#### Step 2 - Harden a container image
Now it's time to build our new image and push it to our Nexus registry.

First, take a look at the container file we will use for the build. Notice that, as part of the container build, we are applying the latest patches as well as executing some security hardening scripts. These scripts are taken from the US DoD Iron Bank project, which aims to provide hardened container images.
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

You just hardened an Open Liberty container image sourced from Dockerhub and uploaded the result to your private container image registry. You can view the uploaded image by visiting the Nexus URL (see Etherpad links), selecting `Browse` in the left pane, and then selecting `registry`.

While the Nexus Registry does not perform container image vulnerability scans, the Red Hat Quay image registry does. To illustrate the difference in before and after vulnerability scan results, images have been uploaded to a publicly-viewable Red Hat Quay repository. Head to [quay.io/repository/mparkins/openliberty](https://quay.io/repository/mparkins/openliberty?tab=tags) to check out the results.

#### Step 3 - Create a custom application image
In this step, we will use the hardened image we just created to optimise an existing Spring application for running on Liberty. The resulting optimised application will be packaged into a container using the same image and pushed to our container image registry.

The container build file has already been provided (`~/dockerfiles/Dockerfile.spring-tomcat-to-liberty`). Before moving on, take a second to familiarise yourself with the contents of this build file. Notice the following:
- This is a multi-stage build, meaning we will run some of the steps in one container and then use the output to build another container. A multi-stage build allows us to cut down on the size and number of layers in our final container image
- There is a variable named `IMAGE` that defines our builder image. We will override the default variable with our hardened image
- There is a variable named `TARGET_JAR` that defines the location of our prebuilt application JAR file. We will override the default variable with a prebuilt Petclinic application JAR file (already uploaded to Nexus for you)
- In the first stage of the build we are running a Liberty utility named `springBootUtility` to create our thin JAR
- The second stage of the build places the thin app and extracted libraries into the relevant Open Liberty directories, ensuring permissions are correct

The existing application (fat JAR) has been uploaded already to `${NEXUS_URL}/repository/maven-releases/com/spring-petclinic/v1.0.0/spring-petclinic-v1.0.0.jar`.

Let's go ahead and kick off the container build:
```bash
export NEXUS_URL=<Nexus URL from Etherpad>

export APP_JAR=${NEXUS_URL}/repository/maven-releases/com/spring-petclinic/v1.0.0/spring-petclinic-v1.0.0.jar

cd ~/dockerfiles

podman build --tag ${REGISTRY_URL}/${USER}/petclinic:v1.0.0 -f Dockerfile.spring-tomcat-to-liberty --build-arg IMAGE=${REGISTRY_URL}/${USER}/openliberty:spring-java8 --build-arg TARGET_JAR=${APP_JAR} .
```

Once complete, we will again push the final image into our registry:
```bash
podman push ${REGISTRY_URL}/${USER}/petclinic:v1.0.0
```

#### Step 4 - Deploy our custom application image
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

_Note:_ If you visit the route and see the Open Liberty splash page, just give it another minute or so. Liberty is still starting up your Petclinic application.

#### Stretch Goal
You should now have two Liberty applications running in the same namespace, although you may have noticed that one has been updated with some additional flair. Set up a new route that splits incoming traffic between the two applications, effectively simulating a blue-green deployment. Name the route whatever you would like and weight the traffic to each application version however you see fit.

__Hint:__ use the OpenShift web console for this one, and make sure you are in the "Administrator" view (i.e. not the "Developer" view).

[Previous Exercise](exercise03) / [Next Exercise](exercise05)
