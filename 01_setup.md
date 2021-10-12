---
layout: page
permalink: /liberty-workshop/setup
---
__Environment Setup__

The instructor should have already provisioned the cluster using the guide here: [Liberty Workshop Setup](https://github.com/mattparko/liberty-workshop-setup)

First up, make sure you have the Etherpad link from the instructor. The Etherpad allows you to pick a username and also has a list of important links and passwords.

All instructions from here on use bash environment variables to reference these links and users. Make sure you substitute them with the proper information whenever you come across them - or add them to your environment variables!

#### Step 1 - Bastion host
The bastion host is a Linux VM that includes all tools required to complete the workshop (like the OpenShift CLI and git tools). It is not mandatory for you to use the bastion host, but it might make things easier and cleaner for you.

Connect to the bastion host using an ssh client (eg. Putty or the built-in ssh client in Windows Subsystem for Linux)
```bash
ssh -l $USER $BASTION_HOST
```

#### Step 2 - OpenShift login
Fire up another browser tab and connect to the OpenShift Web Console.

Enter your username and password.

In the top right, click the username dropdown and select `Copy login command`.

In the newly opened tab, click `Display Token`.

Copy the entire `oc login --token=` line and paste it into your bastion host command line. You should see output similar to the below if the login was successful:
```
Logged into "https://api.openshift.example.com:6443" as "user1" using the token provided.

You don't have any projects. You can try to create a new project, by running

    oc new-project <projectname>

Welcome! See 'oc help' to get started.
```

#### Step 3 - OpenShift console navigation
Let's get a little more familiar with navigating around the OpenShift web console. Head back over to the console in your web browser (you can close the login token page if you haven't already).

You should currently be in the Developer view. To confirm, check the left pane - at the top you should see `</> Developer`. OpenShift splits typical tasks for project administrators and developers across different views. This helps to simplify the interface and navigation.

You may notice at the top of the page a message that states `No Projects exist`. This is fine - we will create a project in the next exercise. Click the `Project` dropdown at the very top of the page. This is where and how you will switch between projects in the console. Again, you don't have any projects yet - don't worry!

We'll come back to the Developer view later, once we have deployed an application. For now, let's explore the administrator view.

Click on the `</> Developer` dropdown at the top of the left pane and select `Administrator`. In this view, you will be able to explore resources in more detail, like networking, storage, service accounts, and role bindings. You can view these resources once we create our first application.

At the very top right of the page you will see a question mark (`?`) symbol. Click on this and select `Command line tools`. You will see a page with links to download and install a number of command line clients, including the OpenShift CLI tool (`oc`), the Helm CLI, and the Tekton CLI. You can always visit and revisit this page to make sure you have a compatible set of command line tools for your cluster.

Once ready, move on to the first exercise where we will create our first project and deploy our first Open Liberty application.

[Next Exercise](exercise01)
