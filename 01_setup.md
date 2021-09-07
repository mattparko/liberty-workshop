---
layout: page
permalink: /liberty-workshop/setup
---
### Environment Setup

The instructor should have already provisioned the cluster using the guide here: https://github.com/mattparko/liberty-workshop-setup

First up, make sure you have the Etherpad link from the instructor. The Etherpad allows you to pick a username and also has a list of important links and passwords.

All instructions from here on use the bash environment variable to reference these links and users. Make sure you substitute them with the proper information whenever you come across them - or add them to your environment variables!

#### Step 1 - Bastion Host
The bastion host is a Linux VM that includes all tools required to complete the workshop (like the OpenShift CLI and git tools). It is not mandatory for you to use the bastion host, but it might make things easier and cleaner for you.

Connect to the bastion host using an ssh client (eg. Putty or the built-in ssh client in Windows Subsystem for Linux)
```bash
ssh -l ${USER} ${BASTION_HOST}
```

#### Step 2 - OpenShift Login
Fire up another browser tab and connect to the OpenShift Web Console

Enter your username and password

In the top right, click the username dropdown and select `Copy login command`

In the newly opened tab, click `Display Token`

Copy the entire `oc login --token=` line and paste it into your bastion host command line:
```
[user1@bastion ~]$ oc login --token=sha256~my_ssh_token --server=https://api.openshift.example.com:6443

Logged into "https://api.openshift.example.com:6443" as "user1" using the token provided.

Using project "default".
```

[Next Exercise](exercise01)
