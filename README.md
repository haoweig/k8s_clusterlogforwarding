# Cluster Logging Operator Installation Guide

## Installation Steps

### 1. Install the Operator

Install the Cluster Logging Operator via OperatorHub in the OpenShift Console. This creates a Subscription object that manages the operator's lifecycle.

**Important**: Select "All namespaces" during installation.

### Why All Namespaces?

The operator must be installed with cluster-wide access (All namespaces) because:
- Different teams/tenants can create their own ClusterLogForwarder resources to monitor their specific applications
- Tenants typically don't have access to OpenShift system namespaces (openshift-*)
- The operator needs to watch for ClusterLogForwarder resources across all namespaces to support multi-tenant logging configurations

### Technical Details

When installing via the OpenShift Console GUI, it creates a Subscription object similar to:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-logging
  namespace: openshift-logging
spec:
  channel: "stable"
  name: cluster-logging
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

### 2. Encode HEC Token

Before creating the ClusterLogForwarder, encode your Splunk HTTP Event Collector (HEC) token:
```bash
# Replace with your actual HEC token
echo -n "<YOUR-HEC-TOKEN>" | base64
```
### 3. Configure Logging

#### Create Secret with HEC Token
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: splunk-hec-secret
  namespace: openshift-logging
type: Opaque
data:
  token: <BASE64-ENCODED-HEC-TOKEN>
```
#### Test HEC token with Splunk
```bash
# Test HEC token with Splunk
curl -k -H "Authorization: Splunk <YOUR-HEC-TOKEN>" \
    -H "Content-Type: application/json" \
    -d '{"event": "Hello, this is a test log 2", "sourcetype": "manual", "index": "<YOUR-SPLUNK-INDEX>"}' \
    https://<YOUR-SPLUNK-HEC-ENDPOINT>/services/collector/event

curl -k -H "Authorization: Splunk <YOUR-HEC-TOKEN>" -H "Content-Type: application/json" -d '{"event": "Hello, this is a test log 2", "sourcetype": "manual", "index": "<YOUR-SPLUNK-INDEX>"}' https://<YOUR-SPLUNK-HEC-ENDPOINT>/services/collector/event
```
Check if HEC has ssl enabled if it fails

### Configure ClusterLogForwarder
This uses the secret created before. We need to create the service account referenced below
```yaml
# ClusterLogForwarder
apiVersion: observability.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance-gitops
  namespace: openshift-logging
spec:
  serviceAccount:
    name: logging-sa
  outputs:
    - name: splunk-hec
      type: splunk
      splunk:
        index: <YOUR-SPLUNK-INDEX>
        url: https://<YOUR-SPLUNK-HEC-ENDPOINT>
        authentication:
          token:
            key: token
            secretName: splunk-hec-secret
      tls:
        insecureSkipVerify: false  # Set to true if using self-signed certificates
  inputs:
    - name: my-namespace-logs
      type: application
      application:
        namespaces:
          - <YOUR-TARGET-NAMESPACE>  # Namespace to collect logs from
  pipelines:
    - name: send-to-splunk
      inputRefs:
        - my-namespace-logs
      outputRefs:
        - splunk-hec
```

### Configure ServiceAccount
```yaml
### ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: logging-sa
  namespace: openshift-logging
###########################
###########################
### ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: application-logs-binding
subjects:
- kind: ServiceAccount
  name: logging-sa
  namespace: openshift-logging
roleRef:
  kind: ClusterRole
  name: collect-application-logs
  apiGroup: rbac.authorization.k8s.io

```

