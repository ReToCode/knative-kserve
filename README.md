# OpenShift + KServe + Knative

## Prerequisites
* An OpenShift cluster

## Installation
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
oc apply -f serverless/knativeserving.yaml

# Create the Knative gateways
oc apply -f serverless/gateways.yaml

# Install cert-manager operator
oc apply -f cert-manager/operator.yaml

# Install KServe
oc apply -f kserve/kserve.yaml

# Install KServe built-in serving runtimes
oc wait --for=condition=ready pod -l control-plane=kserve-controller-manager -n kserve --timeout=300s
oc apply -f kserve/kserve-runtimes.yaml

# Add networkpolicies to allow traffic between knative-serving <> kserve
oc apply -f kserve/networkpolicies.yaml
```

## Testing KServe installation
```bash
# Prerequisites
oc apply -f kserve/namespace.yaml
# Allow to run as user 1337 because of https://istio.io/latest/docs/setup/additional-setup/cni/#compatibility-with-application-init-containers
oc adm policy add-scc-to-user anyuid -z default -n kserve-demo
```

### From google bucket
From https://kserve.github.io/website/0.10/get_started/first_isvc/#2-create-an-inferenceservice
```bash
oc apply -f kserve/samples/sklearn-iris.yaml

curl -k https://sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/sklearn-iris:predict -d @./kserve/samples/iris-input.json
{"predictions":[1,1]}% 
```

### From PVC
```bash
oc apply -f kserve/samples/pvc.yaml
oc apply -f kserve/samples/pvc-pod.yaml
oc cp kserve/samples/model.joblib model-store-pod:/pv/model.joblib -c model-store -n kserve-demo
oc delete -f model-store-pod -n kserve-demo
oc apply -f kserve/samples/sklearn-pvc.yaml

curl -k https://sklearn-pvc-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/sklearn-pvc:predict -d @./kserve/samples/iris-input.json 
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
oc apply -f kserve/samples/sklearn-grpc.yaml

export INPUT_PATH=
export PROTO_FILE=kserve/samples/grpc_predict_v2.proto

grpcurl -insecure -proto $PROTO_FILE  sklearn-grpc-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com:443 inference.GRPCInferenceService.ServerReady
{
  "ready": true
}

grpcurl -insecure -proto $PROTO_FILE -d @ sklearn-grpc-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com:443 inference.GRPCInferenceService.ModelInfer <<< $(cat kserve/samples/input-grpc.json)

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
