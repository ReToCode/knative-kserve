# OpenShift + KServe + Knative

## Prerequisites
* An OpenShift cluster
## Installation with Kourier
```bash
# Install OpenShift Serverless operator
oc apply -f serverless/operator.yaml

# Create an Knative instance
oc apply -f serverless/knativeserving-kourier.yaml

# Install cert-manager operator
oc apply -f cert-manager/operator.yaml

# Install KServe
oc apply -f kserve/kserve.yaml

# Config changes according to https://kserve.github.io/website/0.10/admin/serverless/kourier_networking/#install-kourier-networking-layer
oc edit configmap/inferenceservice-config --namespace kserve
ingress : |- {
    "disableIstioVirtualHost": true
}

# Restart the kserve controller
oc rollout restart deployment kserve-controller-manager -n kserve

# Install KServe built-in serving runtimes
oc wait --for=condition=ready pod -l control-plane=kserve-controller-manager -n kserve --timeout=300s
oc apply -f kserve/kserve-runtimes.yaml
```

## Installation with Istio + Mesh
```bash
# Install Service Mesh operators
oc apply -f service-mesh/operators.yaml

# Create an istio instance
oc apply -f service-mesh/namespace.yaml
oc apply -f service-mesh/smcp.yaml
oc apply -f service-mesh/smmr.yaml

# Install OpenShift Serverless operator
oc apply -f serverless/operator.yaml

# Create an Knative instance
oc apply -f serverless/knativeserving-istio.yaml

# Create the Knative gateways
oc apply -f serverless/gateways.yaml

# Install cert-manager operator
oc apply -f cert-manager/operator.yaml

# Install KServe
oc apply -f kserve/kserve.yaml

# Install KServe built-in serving runtimes
oc wait --for=condition=ready pod -l control-plane=kserve-controller-manager -n kserve --timeout=300s
oc apply -f kserve/kserve-runtimes.yaml

# Add NetworkPolicies to allow traffic to kserve webhook
oc apply -f kserve/networkpolicies.yaml
```

## Testing KServe installation
```bash
# Prerequisites (only relevant for istio)
oc apply -f kserve/namespace.yaml
# Allow to run as user 1337 because of https://istio.io/latest/docs/setup/additional-setup/cni/#compatibility-with-application-init-containers
oc adm policy add-scc-to-user anyuid -z default -n kserve-demo
```

### From google bucket
From https://kserve.github.io/website/0.10/get_started/first_isvc/#2-create-an-inferenceservice
```bash
# Istio
oc apply -f kserve/samples/istio/sklearn-iris.yaml
# Kourier
oc apply -f kserve/samples/kourier/sklearn-iris.yaml

curl -k https://sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/sklearn-iris:predict -d @./kserve/samples/input-iris.json
{"predictions":[1,1]}% 
```

### From PVC
```bash
# Preparation
oc apply -f kserve/samples/pvc.yaml
oc apply -f kserve/samples/pvc-pod.yaml
oc cp kserve/samples/model.joblib model-store-pod:/pv/model.joblib -c model-store -n kserve-demo
oc delete pod -f model-store-pod -n kserve-demo

# Istio
oc apply -f kserve/samples/istio/sklearn-pvc.yaml
# Kourier
oc apply -f kserve/samples/kourier/sklearn-pvc.yaml

curl -k https://sklearn-pvc-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/sklearn-pvc:predict -d @./kserve/samples/input-iris.json 
{"predictions":[1,1]}% 
```

### Using GRPC
**Prerequisites**
```bash
# Enable http2 on OCP ingress to use GRPC against OCP routes
oc annotate ingresses.config/cluster ingress.operator.openshift.io/default-enable-http2=true
# Wait until ingress rollout is completed
oc get po -n openshift-ingress -w
```

From https://kserve.github.io/website/0.10/modelserving/v1beta1/triton/torchscript/#run-a-prediction-with-grpcurl
```bash
# Istio
oc apply -f kserve/samples/istio/torchscript-grpc.yaml
# Kourier
oc apply -f kserve/samples/kourier/torchscript-grpc.yaml

export PROTO_FILE=kserve/samples/grpc_predict_v2.proto
grpcurl -insecure -proto $PROTO_FILE  torchscript-grpc-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com:443 inference.GRPCInferenceService.ServerReady
{
  "ready": true
}

grpcurl -insecure -proto $PROTO_FILE -d @ torchscript-grpc-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com:443 inference.GRPCInferenceService.ModelInfer <<< $(cat kserve/samples/input-grpc.json)

{
  "modelName": "cifar10",
  "modelVersion": "1",
  "outputs": [
    {
      "name": "OUTPUT__0",
      "datatype": "FP32",
      "shape": [
        "1",
        "10"
      ]
    }
  ],
  "rawOutputContents": [
    "viwGwNpLDL7icgK/dusyQAaAD79/KP8/IX2QP4fAs7+HuRk/1+oHwA=="
  ]
}
```

### Canary Deployment
```bash
# Istio
oc apply -f kserve/samples/istio/sklearn-iris.yaml
# Kourier
oc apply -f kserve/samples/kourier/sklearn-iris.yaml

oc get isvc sklearn-iris -n kserve-demo

NAME           URL                                                                                          READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION            AGE
sklearn-iris   http://sklearn-iris-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com   True           100                              sklearn-iris-predictor-00001   109s

# Canary rollout
## Istio
oc apply -f kserve/samples/istio/sklearn-iris-v2.yaml
## Kourier
oc apply -f kserve/samples/kourier/sklearn-iris-v2.yaml

oc get isvc sklearn-iris -n kserve-demo

NAME           URL                                                                                          READY   PREV   LATEST   PREVROLLEDOUTREVISION          LATESTREADYREVISION            AGE
sklearn-iris   http://sklearn-iris-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com   False   90     10       sklearn-iris-predictor-00001   sklearn-iris-predictor-00001   3m40s

curl -k https://sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/sklearn-iris:predict -d @./kserve/samples/input-iris.json
{"predictions":[1,1]}
```

```bash
# Previous revision
oc get isvc sklearn-iris -n kserve-demo -ojsonpath="{.status.components.predictor}"  | jq

{
  "address": {
    "url": "http://sklearn-iris-predictor.kserve-demo.svc.cluster.local"
  },
  "latestCreatedRevision": "sklearn-iris-predictor-00002",
  "latestReadyRevision": "sklearn-iris-predictor-00002",
  "latestRolledoutRevision": "sklearn-iris-predictor-00001",
  "traffic": [
    {
      "latestRevision": true,
      "percent": 10,
      "revisionName": "sklearn-iris-predictor-00002"
    },
    {
      "latestRevision": false,
      "percent": 90,
      "revisionName": "sklearn-iris-predictor-00001",
      "tag": "prev",
      "url": "https://prev-sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com"
    }
  ],
  "url": "http://sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com"
}

# Curl the previous revision
curl -k https://prev-sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/sklearn-iris:predict -d @./kserve/samples/input-iris.json
{"predictions":[1,1]}
```

```bash
# Tag based routing
## Istio
oc apply -f kserve/samples/istio/sklearn-iris-tag-based.yaml
## Kourier
oc apply -f kserve/samples/kourier/sklearn-iris-tag-based.yaml

oc get isvc sklearn-iris -n kserve-demo -ojsonpath="{.status.components.predictor}"  | jq

{
  "address": {
    "url": "http://sklearn-iris-predictor.kserve-demo.svc.cluster.local"
  },
  "latestCreatedRevision": "sklearn-iris-predictor-00003",
  "latestReadyRevision": "sklearn-iris-predictor-00003",
  "latestRolledoutRevision": "sklearn-iris-predictor-00001",
  "traffic": [
    {
      "latestRevision": true,
      "percent": 10,
      "revisionName": "sklearn-iris-predictor-00003",
      "tag": "latest",
      "url": "https://latest-sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com"
    },
    {
      "latestRevision": false,
      "percent": 90,
      "revisionName": "sklearn-iris-predictor-00001",
      "tag": "prev",
      "url": "https://prev-sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com"
    }
  ],
  "url": "http://sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com"
}

# Curl the latest revision
curl -k https://latest-sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/sklearn-iris:predict -d @./kserve/samples/input-iris.json
{"predictions":[1,1]}

# Curl the previous revision
curl -k https://prev-sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/sklearn-iris:predict -d @./kserve/samples/input-iris.json
{"predictions":[1,1]}%
```

### Batching
Example is using https://kserve.github.io/website/0.10/modelserving/batcher/batcher/

```bash
# Istio
oc apply -f kserve/samples/istio/torchserve-batch.yaml
# Kourier
oc apply -f kserve/samples/kourier/torchserve-batch.yaml

# Basic request
curl -k https://torchserve-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/mnist:predict -d @./kserve/samples/input-mnist.json

# Use hey to create some parallel requests
hey -z 10s -c 5 -m POST -D ./kserve/samples/input-mnist.json https://torchserve-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/mnist:predict

Summary:
  Total:        10.3765 secs
  Slowest:      0.7197 secs
  Fastest:      0.5339 secs
  Average:      0.5461 secs
  Requests/sec: 9.1553

  Total data:   7695 bytes
  Size/request: 81 bytes

Response time histogram:
  0.534 [1]     |
  0.552 [89]    |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.571 [0]     |
  0.590 [0]     |
  0.608 [0]     |
  0.627 [0]     |
  0.645 [0]     |
  0.664 [0]     |
  0.683 [0]     |
  0.701 [0]     |
  0.720 [5]     |■■


Latency distribution:
  10% in 0.5353 secs
  25% in 0.5357 secs
  50% in 0.5366 secs
  75% in 0.5374 secs
  90% in 0.5379 secs
  95% in 0.7197 secs
  0% in 0.0000 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0096 secs, 0.5339 secs, 0.7197 secs
  DNS-lookup:   0.0073 secs, 0.0000 secs, 0.1380 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0001 secs
  resp wait:    0.5364 secs, 0.5338 secs, 0.5385 secs
  resp read:    0.0000 secs, 0.0000 secs, 0.0001 secs

Status code distribution:
  [200] 95 responses
```