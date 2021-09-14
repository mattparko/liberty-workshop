---
layout: page
permalink: /liberty-workshop/exercise04
---
__Exercise 4__

Tomcat to Liberty + Container Customisation

In this exercise we will spend some time exploring, building, and customising container images.

Once again we'll use the Petclinic application, though of course with another subtle twist. This time, we will take a prebuilt application (a fat JAR with the Tomcat server built in), optimise it for running on Liberty, and bake it into an image ready for deployment. While we do this, we'll also apply some custom security hardening to our builder image.

In this exercise we will:
1. Apply patches and custom security fixes to an existing container image
1. 
1. 

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
podman build -t registry-nexus.apps.cluster-2xfj7.2xfj7.sandbox135.opentlc.com/${USER}/openliberty:spring-java8 -f Dockerfile.custom-spring-ubi .

podman images
```

Finally, push your built image into the Nexus image registry. We'll use this image in a later step:
```bash
podman push registry-nexus.apps.cluster-2xfj7.2xfj7.sandbox135.opentlc.com/${USER}/openliberty:spring-java8
```

While the Nexus Registry does not perform container scans, the Red Hat Quay image registry does. I pushed both the before and after images into my personal Quay repository to illustrate the difference in scan results. Head to https://quay.io/repository/mparkins/openliberty?tab=tags to check it out.

#### Step 3






[Previous Exercise](exercise03) / [Next Exercise](exercise05)
