# PoC: KServe raw deployment mode

Based on https://kserve.github.io/website/0.10/admin/kubernetes_deployment/

> 📝  The comment in KServe is relevant:
> While both the KServe controller and ModelMesh controller will reconcile InferenceService resources, 
> the ModelMesh controller will only handle those InferenceServices with the 
> serving.kserve.io/deploymentMode: ModelMesh annotation. Otherwise, the KServe controller will handle 
> reconciliation. Likewise, the KServe controller will not reconcile an InferenceService with 
> the serving.kserve.io/deploymentMode: ModelMesh annotation, and will defer under the assumption 
> that the ModelMesh controller will handle it.
> From: https://github.com/kserve/modelmesh-serving/blob/main/docs/quickstart.md#2-deploy-a-model


## Setup

### Basic setup

Install KServe, OSSM and OpenShift Serverless according to the [README](./README.md#installation-with-istio--mesh) and also do [Prerequisites](./README.md#prerequisites-1)

### Additional changes

> ⛔️ Note: you need to configure the `ingressDomain` in `kserve/kserve-config-patch-rawdeployment.yaml` to your cluster's domain.

* Set the deployment mode to `RawDeployment` instead of Serverless
* And use the OpenShift ingress-class `openshift-default`
* And set the ingressDomain to the `OpenShift DNS`
* And set the urlScheme to `https`

```bash
oc apply -f kserve/kserve-config-patch-rawdeployment.yaml
cat kserve/kserve-config-patch-rawdeployment.yaml | grep ingressDomain
```
```text
"ingressDomain"  : "apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com",
```

```bash
# Drop the PeerAuthentication resources because OpenShift routing is not calling the service via istio-ingressgateway and without mTLS
oc delete -f service-mesh/peer-authentication.yaml

# Allow OpenShift Router to talk to our demo namespace directly
oc apply -f kserve/networkpolicies-rawdeployment.yaml
```

## Deploy and test an Inference Service on KServe as raw deployment

> 📝 Note: KServe creates `Ingress` objects for http. As our default OpenShift only allows https routes, we need an additional annotation on the `Route` object for routing to work: `route.openshift.io/termination: "edge"` .
> The examples below already have that additional annotation.

```bash
oc apply -f kserve/samples/istio-raw/sklearn-iris.yaml
```

Which creates

```bash
kubectl tree inferenceservices sklearn-iris -n kserve-demo

NAMESPACE    NAME                                               READY  REASON  AGE
kserve-demo  InferenceService/sklearn-iris                      True           50s
kserve-demo  ├─Deployment/sklearn-iris-predictor                -              45s
kserve-demo  │ └─ReplicaSet/sklearn-iris-predictor-58b98bd768   -              45s
kserve-demo  │   └─Pod/sklearn-iris-predictor-58b98bd768-49js4  True           43s
kserve-demo  ├─HorizontalPodAutoscaler/sklearn-iris-predictor   -              45s
kserve-demo  ├─Ingress/sklearn-iris                             -              9s
kserve-demo  │ ├─Route/sklearn-iris-n84d8                       -              9s
kserve-demo  │ └─Route/sklearn-iris-nljdc                       -              9s
kserve-demo  └─Service/sklearn-iris-predictor                   -              45s
kserve-demo    └─EndpointSlice/sklearn-iris-predictor-ffvjr     -              45s
```

And can be called using:
```bash
oc get route -n kserve-demo
NAME                 HOST/PORT                                                                                       PATH   SERVICES                 PORT                     TERMINATION     WILDCARD
sklearn-iris-fkzk2   sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com   /      sklearn-iris-predictor   sklearn-iris-predictor   edge/Redirect   None
sklearn-iris-qb8cf   sklearn-iris-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com             /      sklearn-iris-predictor   sklearn-iris-predictor   edge/Redirect   None
```

```bash
curl -k https://sklearn-iris-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/sklearn-iris:predict -d @./kserve/samples/input-iris.json

{"predictions":[1,1]}% 

curl -k https://sklearn-iris-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com/v1/models/sklearn-iris:predict -d @./kserve/samples/input-iris.json

{"predictions":[1,1]}% 
```

## Deploy and test an Inference Service with GRPC

> ⛔️ Note: this does currently not work upstream. I created an issue: https://github.com/kserve/kserve/issues/2961 

```bash
oc apply -f kserve/samples/istio-raw/torchscript-grpc.yaml

export PROTO_FILE=kserve/samples/grpc_predict_v2.proto
grpcurl -insecure -proto $PROTO_FILE  torchscript-grpc-predictor-kserve-demo.apps.rlehmann-ocp-4-12.serverless.devcluster.openshift.com:443 inference.GRPCInferenceService.ServerReady

# DOES CURRENTLY NOT WORK
```

## Setup ModelMesh alongside

```bash
RELEASE=release-0.10
git clone -b $RELEASE --depth 1 --single-branch https://github.com/kserve/modelmesh-serving.git
cd modelmesh-serving

oc create namespace modelmesh-serving
./scripts/install.sh --namespace-scope-mode --namespace modelmesh-serving --quickstart
cd ..
```

## Deploy and test an Inference Service on ModelMesh

```bash
oc apply -f modelmesh/samples/sklearn.yaml
```

```text
oc get -n modelmesh-serving pod
NAME                                             READY   STATUS    RESTARTS   AGE
etcd-7d7db5bb8f-spnql                            1/1     Running   0          11m
minio-676868b987-4bp8g                           1/1     Running   0          11m
modelmesh-controller-87db89d44-bc6cd             1/1     Running   0          10m
modelmesh-serving-mlserver-0.x-8d7d594d7-dpl4x   4/4     Running   0          5m13s
modelmesh-serving-mlserver-0.x-8d7d594d7-w44cq   4/4     Running   0          5m13s
```

The KServe controller is ignoring the ModelMesh `InferenceService` 
```text
manager {"level":"info","ts":1685611623.0801382,"logger":"v1beta1Controllers.InferenceService","msg":"Inference service deployment mode ","deployment mode ":"ModelMesh"}
manager {"level":"info","ts":1685611623.08016,"logger":"v1beta1Controllers.InferenceService","msg":"Skipping reconciliation for InferenceService","serving.kserve.io/deploymentMode":"ModelMesh","apiVersion":"serving.kserve.io/v1beta1","isvc":"example-sklearn-isvc"}
```
