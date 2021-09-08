---
layout: page
permalink: /liberty-workshop/exercise02
---
__Exercise 2__

You should now have a running application that is being managed by the Open Liberty Operator.

We will now start to explore some additional management and troubleshooting features like monitoring, as well as the Open Liberty Operator tracing and dumping capabilities.

In this exercise we will:
1. Explore the out-of-the-box monitoring and dashboards
1. Create and export an Open Liberty dump
1. Create and export an Open Liberty trace

#### Step 1
Let's kick things off by generating some load against out application. The following commands will fetch the application's route name (URL) and then execute 1000 `curl` requests against it:
```bash
export APP_ROUTE=$(oc get route demo-app -o json | jq -r ".spec.host")

for (( i=1; i<=1000; i++ )); do curl -s $APP_ROUTE > /dev/null; done
```

#### Step 2
Now let's check out the monitoring dashboards in OpenShift.

Navigate to the Web Console and select the Developer's perspective (i.e. not the Administrator perspective) in the left pane. If you don't see your application, make sure you have the correct project selected (top left of the main pane).

In the left pane, select Monitoring. Explore the various dashboards and options, like Time Range and Refresh Interval.

Click on a dashboard (or select the Metrics tab) to get more detailed views. Click on Show PromQL to see the Prometheus query used for metric collection. You can even temporarily edit the query and re-run it.

#### Step 3


#### Stretch Goal


[Previous Exercise](exercise01) / [Next Exercise](exercise03)
