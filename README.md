# Loki-Grafana Integration
**Prerequisites:**

**i. S3 storage**

**ii. ROSA/ARO/OCP 4.11 cluster**

**iii. htpasswd utility install (sudo dnf install httpd-tools)**

## Install the Red Hat OpenShift Logging operator
1. Create the openshift-logging namespace if needed
~~~
oc get project openshift-logging >/dev/null 2>&1 || oc new-project openshift-logging
~~~
2. Create the operator group if needed
~~~
oc get -n openshift-logging og >/dev/null 2>&1 || oc create -f config/openshift-logging-operator-group.yaml
~~~
3. Create the operator subscription to install the operator from the OperatorHub
~~~
oc create -f config/openshift-logging-operator-subscription.yaml
~~~
4. Wait until the operator pod is running
~~~
oc get pods -w -n openshift-logging
~~~

## Install the Loki operator
1. Create CRD for ES
~~~
oc create -f config/logging.openshift.io_elasticsearches.yaml
~~~
2. Create the operator group if needed
~~~
oc get -n openshift-logging og >/dev/null 2>&1 || oc create -f config/openshift-operators-redhat-operator-group.yaml
~~~
3. Create the operator subscription to install the operator from the OperatorHub
~~~
oc create -f config/loki-operator-subscription.yaml
~~~
4. Wait until the operator pod is running
~~~
oc get pods -w -n openshift-operators-redhat
~~~

## Configure Loki
1. Create the S3 bucket in AWS using your preferred method
2. Set environment variables
~~~
oc project openshift-logging
export AWS_ACCESS_KEY_ID=<your_aws_access_key_id>
export AWS_ACCESS_KEY_SECRET=<your_aws_access_key_secret>
export AWS_REGION=<aws_region>
export AWS_S3_BUCKET=<your_aws_s3_bucket_name>
~~~
3. Create secret for S3
~~~
cat config/loki-s3-secret.yaml | envsubst | oc create -f -
~~~
4. Cleanup the environment
~~~
unset AWS_ACCESS_KEY_ID AWS_ACCESS_KEY_SECRET AWS_REGION AWS_S3_BUCKET
~~~
5. Create LokiStack custom resource
~~~
oc create -f config/logging-loki.yaml
~~~
6. Create ClusterLogging custome resource
~~~
oc create -f config/clusterlogging.yaml
~~~
7. Verify all the pods are running
~~~
oc get pods -n openshift-logging
~~~
8. Verify the logs from OpenShift Console (Refresh Webconsole --> Observe --> Logs) 

## Install the Grafana operator
1. Create the operator subscription to install the operator from the OperatorHub
~~~
oc create -f config/grafana-operator-subscription.yaml
~~~
2. Wait until the operator pod is running
~~~
oc get pods -n openshift-logging -w
~~~

## Configure Grafana
1. Create the Grafana secrets
~~~
oc create -f config/grafana-cr-proxy-secret.yaml
oc create -f config/grafana-cr-htpasswd-secret.yaml
oc create -f config/grafana-cr-creds-secret.yaml
~~~
2. Create the Grafana custom resource
~~~
oc create -f config/grafana-cr.yaml
~~~
3. Wait until the Grafana pods are are running
~~~
oc get pods -w -n openshift-logging
~~~
4. Add the role to the Grafana serviceaccount
~~~
oc adm policy add-cluster-role-to-user cluster-monitoring-view -z grafana-serviceaccount
~~~

## Add Grafana data sources (Prometheus and Loki)
1. Set the BEARER_TOKEN environment variable from the grafana serviceaccount secret token
~~~
SA=grafana-serviceaccount
SECRET=$(oc get sa $SA -o jsonpath='{.secrets[0].name}')
export BEARER_TOKEN=$(oc get secret $SECRET -o jsonpath='{.metadata.annotations.openshift\.io/token-secret\.value}')
~~~
2. Create the data sources
~~~
cat config/grafana-datasources.yaml | envsubst | oc create -f -
~~~
3. Cleanup the environment
~~~
unset SA SECRET BEARER_TOKEN
~~~

## Login to the Grafana console
1. Using the OpenShift console in your browser, select the openshift-logging project
2. Navigate to Networking --> Routes
3. Open the grafana-route location
4. Login using your OpenShift console credentials