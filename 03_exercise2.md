---
layout: page
permalink: /liberty-workshop/exercise02
---
__Exercise 2 - Open Liberty observability__

You should now have a running application that is being managed by the Open Liberty operator.

We will now start to explore some additional management and troubleshooting features like monitoring, as well as the Open Liberty operator tracing and dumping capabilities.

In this exercise we will:
1. Explore the out-of-the-box monitoring and dashboards
1. Create an Open Liberty dump using the operator
1. Create an Open Liberty trace using the operator

#### Step 1 - Generate application load
Let's kick things off by generating some load against out application. The following commands will fetch the application's route name (URL) and then execute 1000 `curl` requests against it:
```bash
export APP_ROUTE=$(oc get route demo-app -o json | jq -r ".spec.host")

for (( i=1; i<=1000; i++ )); do curl -s $APP_ROUTE > /dev/null; done
```

#### Step 2 - View the dashboards
Now let's check out the monitoring dashboards in OpenShift.

Navigate to the Web Console and select the Developer's perspective (i.e. not the Administrator perspective) in the left pane. If you don't see your application, make sure you have the correct project selected (top left of the main pane).

In the left pane, select Monitoring. Explore the various dashboards and options, like Time Range and Refresh Interval.

Click on a dashboard (or select the Metrics tab) to get more detailed views. Click on Show PromQL to see the Prometheus query used for metric collection. You can even temporarily edit the query and re-run it.

#### Step 3 - Create a Liberty dump
We will now create and export a Liberty dump with the assistance of the Open Liberty operator.

First up, get the application pod name and create environment variables:
```bash
export PODNAME=$(oc get pod | grep -i demo-app | awk '{print $1}')

export NAMESPACE=$USER
```

Now create a new Open Liberty Dump resource using the operator:
```bash
echo "apiVersion: openliberty.io/v1beta1
kind: OpenLibertyDump
metadata:
  name: example-dump
  labels:
    app: frontend
spec:
  include:
    - heap
    - thread
    - system
  podName: $PODNAME" | oc apply -n $NAMESPACE -f -
```

#### Step 4 - Export the Liberty dump
Once the Liberty dump has been created, copy it out of the pod onto local storage. The dump only takes a few seconds to complete in this sample application, but you can check the status by running:
```bash
oc get openlibertydump example-dump -o yaml
```

You should see a status entry with `type: Completed` as well as the dump file location / path.

You can export the dump file from the pod like so:
```bash
oc cp $NAMESPACE/$PODNAME:/serviceability/$NAMESPACE/$PODNAME .
```

Feel free to unpack the dump file and explore it:
```bash
mkdir dump

unzip *.zip -d dump/

ls -l dump/
```

Clean up by deleting the Open Liberty Dump resource and the dump file:
```bash
oc delete openlibertydump example-dump

oc exec $PODNAME -- rm -rf /serviceability/$NAMESPACE/$PODNAME
```

#### Step 5 - Create a Liberty trace
Now let's create a Liberty trace with the help of the operator. This is very similar to the steps taken to create a Liberty dump, so let's get straight into it:
```bash
# Create the trace
echo "apiVersion: openliberty.io/v1beta1
kind: OpenLibertyTrace
metadata:
  name: example-trace
  labels:
    app: frontend
spec:
  podName: $PODNAME
  traceSpecification: '*=info:com.ibm.ws.webcontainer*=all'" | oc apply -n $NAMESPACE -f -

# Validate the trace
oc get openlibertytraces

# See the trace files
oc exec $PODNAME -- ls /serviceability/$NAMESPACE/$PODNAME
```

Read more on observability using the Open Liberty operator in the [upstream documentation](https://github.com/OpenLiberty/open-liberty-operator/blob/master/doc/observability-deployment.adoc)

Once ready, stop the trace by deleting the resource:
```bash
oc delete openlibertytrace example-trace -n $NAMESPACE
```

#### Stretch Goal
You may have noticed we did not export the Liberty trace files. How would you go about doing that? Is it possible to view the files without exporting them?

[Previous Exercise](exercise01) / [Next Exercise](exercise03)
