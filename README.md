# OpenShift + KServe + Knative with Service Mesh

## Prerequisites
* An OpenShift cluster, configured with a reachable domain for OCP routes
* OC client pointing to that cluster

## Installation of Knative with Service Mesh
```bash
# Install Service Mesh operators
oc apply -f service-mesh/operators.yaml
sleep 30
oc wait --for=condition=ready pod -l name=istio-operator -n openshift-operators --timeout=300s
oc wait --for=condition=ready pod -l name=jaeger-operator -n openshift-operators --timeout=300s
oc wait --for=condition=ready pod -l name=kiali-operator -n openshift-operators --timeout=300s

# Create an istio instance
oc apply -f service-mesh/namespace.yaml
oc apply -f service-mesh/smcp.yaml
sleep 15
oc wait --for=condition=ready pod -l app=istiod -n istio-system --timeout=300s
oc wait --for=condition=ready pod -l app=istio-ingressgateway -n istio-system --timeout=300s
oc wait --for=condition=ready pod -l app=istio-egressgateway -n istio-system --timeout=300s

oc create ns kserve
oc create ns kserve-demo
oc create ns knative-serving
oc apply -f service-mesh/smmr.yaml
oc apply -f service-mesh/peer-authentication.yaml # we need this because of https://access.redhat.com/documentation/en-us/openshift_container_platform/4.12/html/serverless/serving#serverless-domain-mapping-custom-tls-cert_domain-mapping-custom-tls-cert

# Install OpenShift Serverless operator
oc apply -f serverless/operator.yaml
sleep 30
oc wait --for=condition=ready pod -l name=knative-openshift -n openshift-serverless --timeout=300s
oc wait --for=condition=ready pod -l name=knative-openshift-ingress -n openshift-serverless --timeout=300s
oc wait --for=condition=ready pod -l name=knative-operator -n openshift-serverless --timeout=300s

# Create a Knative Serving installation
oc apply -f serverless/knativeserving-istio.yaml
sleep 15
oc wait --for=condition=ready pod -l app=controller -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=net-istio-controller -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=net-istio-webhook -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=autoscaler-hpa -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=domain-mapping -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=webhook -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=activator -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=autoscaler -n knative-serving --timeout=300s

# Create the Knative gateways
oc apply -f serverless/gateways.yaml

# Install cert-manager operator
oc apply -f cert-manager/operator.yaml
sleep 60
oc wait --for=condition=ready pod -l app=webhook -n cert-manager --timeout=300s
oc wait --for=condition=ready pod -l app=cainjector -n cert-manager --timeout=300s
oc wait --for=condition=ready pod -l app=cert-manager -n cert-manager --timeout=300s
```

## Installation of KServe using RHODS

```bash
# Install RHODS (or ODH) operator
oc apply -f rhods/operator.yaml
oc wait --for=condition=ready pod -l name=rhods-operator -n redhat-ods-operator  --timeout=300s

# Install KServe using KfDef
oc apply -f rhods/kserve-kfdef.yaml
```

## Creating a serving runtime

> 📝 Note: Make sure that your serving runtime image does not need to run as a root or a specific uid.
> Otherwise `anyuid` is required on the target namespace.

```bash
# Install a runtime that does work without `anyuid`
oc apply -f rhods/serving-runtime.yaml
```

## Testing RHODS installation

```bash
# Deploy an InferenceService
cat <<-EOF | oc apply -f -
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
  namespace: kserve-demo
  annotations:
    sidecar.istio.io/inject: "true"
    sidecar.istio.io/rewriteAppHTTPProbers: "true"
    serving.knative.openshift.io/enablePassthrough: "true"
spec:
  predictor:
    model:
      runtime: kserve-sklearnserver
      modelFormat:
        name: sklearn
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model"
EOF
```

```bash
# Testing it
oc get isvc -n kserve-demo
NAME           URL                                                                                          READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION            AGE
sklearn-iris   http://sklearn-iris-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com   True           100                              sklearn-iris-predictor-00001   2m14s

curl -k https://sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/sklearn-iris:predict -d @./kserve/samples/input-iris.json
{"predictions":[1,1]}
```


## Testing vanilla KServe installation

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
oc delete pod --force model-store-pod -n kserve-demo

# Istio
oc apply -f kserve/samples/istio/sklearn-pvc.yaml
# Kourier
oc apply -f kserve/samples/kourier/sklearn-pvc.yaml

curl -k https://sklearn-pvc-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/sklearn-pvc:predict -d @./kserve/samples/input-iris.json 
{"predictions":[1,1]}% 
```

### Using GRPC
* From https://kserve.github.io/website/0.10/modelserving/v1beta1/triton/torchscript/#run-a-prediction-with-grpcurl
 
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
* From https://kserve.github.io/website/0.10/modelserving/batcher/batcher/

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


### Custom transformer (has service to service communication)
*  From https://kserve.github.io/website/0.10/modelserving/v1beta1/transformer/torchserve_image_transformer/

```bash
# Istio
oc apply -f kserve/samples/istio/torch-transformer.yaml
# Kourier
oc apply -f kserve/samples/kourier/torch-transformer.yaml

# Two Knative Services are created
oc get ksvc -n kserve-demo
NAME                            URL                                                                                                            LATESTCREATED                         LATESTREADY                           READY   REASON
torch-transformer-predictor     https://torch-transformer-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com     torch-transformer-predictor-00002     torch-transformer-predictor-00002     True
torch-transformer-transformer   https://torch-transformer-transformer-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com   torch-transformer-transformer-00001   torch-transformer-transformer-00001   True

# Run prediction
curl -k https://torch-transformer-transformer-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/mnist:predict -d @./kserve/samples/input-image.json

{"predictions":[[2]]}%

# In this example the transformer communicates with the predictor:
oc get po -n kserve-demo torch-transformer-transformer-00001-deployment-56db8449cb-wlcbp -o yaml | grep args -A 6
- args:
- --model_name
- mnist
- --predictor_host
- torch-transformer-predictor.kserve-demo
- --http_port
- "8080"

# So it will actually communicate with the DNS `torch-transformer-predictor.kserve-demo` and reach
# 1) with kourier: the cluster-local gateway and go through Kourier
oc get svc -n kserve-demo torch-transformer-predictor
NAME                          TYPE           CLUSTER-IP   EXTERNAL-IP                                                  PORT(S)   AGE
torch-transformer-predictor   ExternalName   <none>       kourier-internal.knative-serving-ingress.svc.cluster.local   80/TCP    13m

# 2) with istio: bypass the cluster-local gateway and go directly through the mesh
oc get svc -n kserve-demo torch-transformer-predictor

NAME                          TYPE           CLUSTER-IP   EXTERNAL-IP                                            PORT(S)   AGE
torch-transformer-predictor   ExternalName   <none>       knative-local-gateway.istio-system.svc.cluster.local   80/TCP    29s

# bypassed because of the mesh virtual-services:
oc get virtualservice -n kserve-demo | grep mesh
NAME                                    GATEWAYS  HOSTS                                                                                                                                                                                                                                                                AGE
torch-transformer-predictor-mesh        ["mesh"]  ["torch-transformer-predictor.kserve-demo","torch-transformer-predictor.kserve-demo.svc","torch-transformer-predictor.kserve-demo.svc.cluster.local"]                                                                                                                89s
torch-transformer-transformer-mesh      ["mesh"]  ["torch-transformer-transformer.kserve-demo","torch-transformer-transformer.kserve-demo.svc","torch-transformer-transformer.kserve-demo.svc.cluster.local"]                                                                                                          2m
```

### Inference Graph (has service to service communication)
* From https://kserve.github.io/website/0.10/modelserving/inference_graph/image_pipeline/
* This is currently broken in KServe. Wait for https://github.com/kserve/kserve/pull/2830 to be merged to re-test.
* An additional PR was necessary: https://github.com/kserve/kserve/pull/2839

```bash
# Patch the kserve cluster role to allow setting finalizers on `InferenceGraphs`. Otherwise we end up with
# "error":"services.serving.knative.dev \"dog-breed-pipeline\" is forbidden: cannot set blockOwnerDeletion if an ownerReference refers to a resource you can't set finalizers on: , <nil>"
# PR to KServe to fix this: https://github.com/kserve/kserve/pull/2839
oc apply -f kserve/kserve-cluster-role-patch.yaml

# Istio
oc apply -f kserve/samples/istio/cat-dog-breed.yaml
oc apply -f kserve/samples/istio/inference-graph.yaml
# Kourier
oc apply -f kserve/samples/kourier/cat-dog-breed.yaml
oc apply -f kserve/samples/kourier/inference-graph.yaml

# Get the URL of the inference graph
oc get ig  dog-breed-pipeline -n kserve-demo
NAME                 URL                                                                                                 READY   AGE
dog-breed-pipeline   https://dog-breed-pipeline-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com   True    24s

# Calling the service
curl -k https://dog-breed-pipeline-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com -d @./kserve/samples/input-cat.json

{"predictions": ["cat"]}% 

# Note for Istio
Internally, it does something similar to our `Domain-Mapping`, so this 
> If you use net-istio for Ingress and enable mTLS via SMCP using security.dataPlane.mtls: true, Service Mesh deploys DestinationRules for the *.local host, which does not allow DomainMapping for OpenShift Serverless.
> To work around this issue, enable mTLS by deploying PeerAuthentication instead of using security.dataPlane.mtls: true.
also applies here. We cannot use `security.dataPlane.mtls: true` with KServe.

# Kourier
TODO: This is currently broken in KServe. Wait for https://github.com/kserve/kserve/pull/2830 to be merged to re-test.
Issue created: https://github.com/kserve/kserve/issues/2856
```

Additional VS that KServe generates for the wrapper service:
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  annotations:
    serving.knative.openshift.io/enablePassthrough: "true"
    sidecar.istio.io/inject: "true"
    sidecar.istio.io/rewriteAppHTTPProbers: "true"
  labels:
    serviceEnvelope: kserve
  name: cat-dog-classifier
  namespace: kserve-demo
  ownerReferences:
  - apiVersion: serving.kserve.io/v1beta1
    blockOwnerDeletion: true
    controller: true
    kind: InferenceService
    name: cat-dog-classifier
    uid: 61182a3a-376a-4db3-a8b0-a9e403b82903
  uid: 91204ee2-c62f-4eed-be71-a1563502266d
spec:
  gateways:
  - knative-serving/knative-local-gateway
  - knative-serving/knative-ingress-gateway
  hosts:
  - cat-dog-classifier.kserve-demo.svc.cluster.local
  - cat-dog-classifier-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com
  http:
  - headers:
      request:
        set:
          Host: cat-dog-classifier-predictor.kserve-demo.svc.cluster.local
    match:
    - authority:
        regex: ^cat-dog-classifier\.kserve-demo(\.svc(\.cluster\.local)?)?(?::\d{1,5})?$
      gateways:
      - knative-serving/knative-local-gateway
    - authority:
        regex: ^cat-dog-classifier-kserve-demo\.apps\.rlehmann-ocp-4-12\.serverless\.devcluster\.openshift\.com(?::\d{1,5})?$
      gateways:
      - knative-serving/knative-ingress-gateway
    route:
    - destination:
        host: knative-local-gateway.istio-system.svc.cluster.local
        port:
          number: 80 -> 8081 (http2)
      weight: 100
```

### Model Explainability
* From https://kserve.github.io/website/0.10/modelserving/explainer/alibi/moviesentiment/
* Has upstream issues: https://github.com/kserve/kserve/issues/2843
* Does only work with a patch on the kserve images: 
  * https://github.com/kserve/kserve/issues/2844
  * https://github.com/kserve/kserve/pull/2845


```bash
# Istio
oc apply -f kserve/samples/istio/moviesentiment.yaml

# Kourier
oc apply -f kserve/samples/kourier/moviesentiment.yaml

# Predict
curl -k https://moviesentiment-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/moviesentiment:predict -d '{"instances":["a visually flashy but narratively opaque and emotionally vapid exercise ."]}'
{"predictions":[0]}

curl -k https://moviesentiment-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/moviesentiment:predict -d '{"instances":["a touching , sophisticated film that almost seems like a documentary in the way it captures an italian immigrant family on the brink of major changes ."]}'
{"predictions":[1]}

# Explain
curl -k https://moviesentiment-explainer-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/moviesentiment:explain -d '{"instances":["a visually flashy but narratively opaque and emotionally vapid exercise ."]}'
{"meta":{"name":"AnchorText","type":["blackbox"],"explanations":["local"],"params":{"seed":0,"sample_proba":0.5},"version":"0.6.5"},"data":{"anchor":["exercise"],"precision":1.0,"coverage":0.5005,"raw":{"feature":[9],"mean":[1.0],"precision":[1.0],"coverage":[0.5005],"examples":[{"covered_true":["UNK UNK flashy but narratively UNK UNK emotionally vapid exercise UNK","UNK visually flashy but UNK UNK UNK UNK vapid exercise .","UNK visually flashy UNK UNK opaque UNK UNK UNK exercise .","UNK UNK UNK but narratively opaque and emotionally vapid exercise UNK","UNK UNK flashy but narratively UNK and emotionally vapid exercise .","a visually UNK but UNK opaque UNK UNK UNK exercise UNK","a visually UNK UNK narratively UNK UNK UNK UNK exercise .","UNK UNK flashy UNK narratively UNK UNK UNK vapid exercise UNK","a UNK flashy UNK UNK UNK and UNK UNK exercise UNK","a visually UNK but UNK UNK and emotionally vapid exercise ."],"covered_false":[],"uncovered_true":[],"uncovered_false":[]}],"all_precision":0,"num_preds":1000000,"success":true,"names":["exercise"],"positions":[63],"instance":"a visually flashy but narratively opaque and emotionally vapid exercise .","instances":["a visually flashy but narratively opaque and emotionally vapid exercise ."],"prediction":[0]}}}
```
## Testing KServe with Knative Eventing

### Prerequisites

Make sure you followed the instructions in [Testing KServe installation - Prerequisites](#prerequisites-1) first.

Then follow these:
```bash

# Create a Knative Eventing installation
oc apply -f serverless/knativeeventing.yaml

# Create resources for inference logging: broker, trigger, message dumper ksvc
oc apply -f serverless/inference-logger-resources.yaml
```

### Inference logging

```
                   ┌───────────────────┐
                   │                   │
                   │ InferenceService  │
                   │                   │
┌──────────────┐   │   ┌───────────┐   │
│              ├───┼──►│ Predictor │   │        ┌────────────────┐             ┌──────────────────┐
│ Client       │   │   │  Service  │   │        │                ├────────┐    │                  │
│              │◄──┼───┤           │   │   ┌───►│    Broker      │ Trigger├───►│  Event Display   │
└──────────────┘   │   └───────────┘   │   │    │                ├────────┘    │   Service        │
                   │                   │   │    └────────────────┘             │                  │
                   │   ┌───────────┐   │   │                                   └──────────────────┘
                   │   │           │   │   │
                   │   │ Logger    ├───┼───┘
                   │   │           │   │
                   │   └───────────┘   │
                   │                   │
                   │                   │
                   │                   │
                   │                   │
                   └───────────────────┘
```

```bash
# Same as the previous tests, only this time we have inference log events sent to Knative Eventing Broker
$ oc apply -f kserve/samples/istio/sklearn-iris-inference-logging.yaml

$ curl -k https://sklearn-iris-predictor-kserve-demo.apps.aliok-c088.serverless.devcluster.openshift.com/v1/models/sklearn-iris:predict -d @./kserve/samples/input-iris.json
{"predictions":[1,1]}% 

$ k logs -n kserve-demo message-dumper-00001-deployment-7f7874bd66-ssv89

☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 1.0
  type: org.kubeflow.serving.inference.response
  source: http://localhost:9081/
  id: 464d1291-1bbd-4c05-b592-d6ae24de6555
  time: 2023-05-02T14:05:37.874021224Z
  datacontenttype: application/json
Extensions,
  component: predictor
  endpoint:
  inferenceservicename: sklearn-iris
  knativearrivaltime: 2023-05-02T14:05:37.877365133Z
  namespace: kserve-demo
  traceparent: 00-aa63c8db0dff507e246a7abf10a21a7d-d082068f6e5be1d3-00
Data,
  {
    "predictions": [
      1,
      1
    ]
  }
☁️  cloudevents.Event
Validation: valid
Context Attributes,
  specversion: 1.0
  type: org.kubeflow.serving.inference.request
  source: http://localhost:9081/
  id: 464d1291-1bbd-4c05-b592-d6ae24de6555
  time: 2023-05-02T14:05:37.871883206Z
  datacontenttype: application/x-www-form-urlencoded
Extensions,
  component: predictor
  endpoint:
  inferenceservicename: sklearn-iris
  knativearrivaltime: 2023-05-02T14:05:37.878989816Z
  namespace: kserve-demo
  traceparent: 00-f53727be09d26dfb9f7d75bb436a9280-d6b647fe8adc12ff-00
Data,
  {  "instances": [    [6.8,  2.8,  4.8,  1.4],    [6.0,  3.4,  4.5,  1.6]  ]}
```

PRs:
- https://github.com/kserve/kserve/pull/2792

### Drift detection

```
                   ┌───────────────────┐
                   │                   │
                   │ InferenceService  │
                   │                   │
┌──────────────┐   │   ┌───────────┐   │
│              ├───┼──►│ Predictor │   │        ┌────────────────┐             ┌──────────────────┐
│ Client       │   │   │  Service  │   │        │                ├────────┐    │                  │
│              │◄──┼───┤           │   │   ┌───►│    Broker      │ Trigger├───►│  Drift Detector  │
└──────────────┘   │   └───────────┘   │   │    │                ├────────┘    │   Service        │
                   │                   │   │    └────────────────┘             │                  │
                   │   ┌───────────┐   │   │                                   └──────────────────┘
                   │   │           │   │   │
                   │   │ Logger    ├───┼───┘
                   │   │           │   │
                   │   └───────────┘   │
                   │                   │
                   │                   │
                   │                   │
                   │                   │
                   └───────────────────┘
```

See https://github.com/aliok/yaml/tree/main/data-science#drift-detection-on-openshift

PRs:
- https://github.com/kserve/kserve/pull/2787
- https://github.com/kserve/kserve/pull/2888
