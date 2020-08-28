** INTRO **
=========

This is a simple project that shows you how to interact with a Kubernetes cluster usings its API without the kubectl client.

> **Note:** Of course you can work with the ofitial libraries in other languages but the proposal of this doc is doing it simple.
> This is the ofitial link to know how to do it on code: https://kubernetes.io/docs/reference/using-api/client-libraries/

Requirements
=========

To have access to the K8S cluster and to install a usefull tool to work with json:

```bash
sudo apt install jq
```

Method 1: Using a Service Account
========

We can use one SA that already exists or create a new one for that purpose.

Once we have it, we can get all the data we need from it in order to be able to make some queries:

```bash
# Get the ServiceAccount's token Secret's name
SECRET=$(kubectl get serviceaccount ${SERVICE_ACCOUNT} -o json | jq -Mr '.secrets[].name | select(contains("token"))')

# Extract the Bearer token from the Secret and decode
TOKEN=$(kubectl get secret ${SECRET} -o json | jq -Mr '.data.token' | base64 -d)

# Extract, decode and write the ca.crt to a temporary location
kubectl get secret ${SECRET} -o json | jq -Mr '.data["ca.crt"]' | base64 -d > /tmp/ca.crt

# Get the API Server location

APISERVER=https://$(kubectl -n default get endpoints kubernetes --no-headers | awk '{ print $2 }')

```

Now we have all we need to make queries:

```bash
curl -s $APISERVER/openapi/v2  --header "Authorization: Bearer $TOKEN" --cacert /tmp/ca.crt | less
```

Method 2: Proxy
=========

In this method, we are going to use the kubectl client as a reverse proxy. Is much simpler and esier:

```bash
PORT_NUMER=1234
kubectl proxy --port $PORT_NUMBER &
curl -s http://localhost:1234/api/v1 | less
```

Some interesting queries:
==================

```bash

# Check the K8S API:
curl -s http://localhost:1234/api/v1 | less

# Nodes info:
curl -s http://localhost:1234/api/v1/nodes | jq '.items[].metadata.labels'

# Node status:
curl -s http://localhost:1234/api/v1/nodes | jq -rM '(.items[].status.conditions[3].type)'
curl -s http://localhost:1234/api/v1/nodes | jq -rM '.items[] | (.metadata.name) + " " + (.status.conditions[3].type)'

# Pods in a desire namespace:
curl -s http://localhost:1234/api/v1/namespaces/my-app/pods/ | jq -rM '.items[].metadata.name'
curl -s http://localhost:1234/api/v1/namespaces/kube-system/pods/ | jq -rM '.items[].metadata.name'

## Pods
# Pod status
curl -s http://localhost:1234/api/v1/pods |jq -rM '.items[] | (.metadata.name) + " " + (.status.phase)'
curl -s http://localhost:1234/api/v1/pods |jq -rM '.items[] | (.metadata.name) + " " + (.status.phase) + " Reinicios: " + (.status.containerStatuses[0].restartCount|tostring)'

# A concrete Pod status:
curl -s http://localhost:1234/api/v1/namespaces/kube-system/pods/my-ingress-pod
curl -s http://localhost:1234/api/v1/namespaces/kube-system/pods/my-ingress-pod |jq -rM '(.status.containerStatuses[0].restartCount)'
curl -s http://localhost:1234/api/v1/namespaces/kube-system/pods/my-ingress-pod |jq -rM '(.metadata.name) + " " + (.status.phase) + " Reinicios: " + (.status.containerStatuses[0].restartCount|tostring)'

# Cluster consume:
curl -s http://localhost:1234/apis/metrics.k8s.io/v1beta1/nodes |jq -rM '.items[]| (.metadata.name) + "  Uso CPU --> " + (.usage.cpu) + "  Uso Memoria --> " + (.usage.memory)'

# Cluster status:
curl -s http://localhost:1234/healthz/
curl -s http://localhost:1234/etcd/
