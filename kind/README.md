# Setup on Kind

## Prerequisites
* A running kind cluster
* MetalLB installed
* Ability to reach Kubernetes `type: LoadBalancer` IP from your machine


## Setup
```bash
export ISTIO_VERSION=1.15.0
export KNATIVE_VERSION=knative-v1.7.0
export KSERVE_VERSION=v0.10.1
export CERT_MANAGER_VERSION=v1.3.0

# Install istio
kubectl apply -f ./istio-namespace.yaml
istioctl manifest apply -f ./istio-minimal-operator.yaml -y

# Install Knative
kubectl apply --filename https://github.com/knative/serving/releases/download/$KNATIVE_VERSION/serving-crds.yaml
kubectl apply --filename https://github.com/knative/serving/releases/download/$KNATIVE_VERSION/serving-core.yaml
kubectl apply --filename https://github.com/knative/net-istio/releases/download/$KNATIVE_VERSION/release.yaml

# Configure domain
kubectl patch configmap/config-domain \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"10.89.0.200.sslip.io":""}}'

# Install Cert Manager
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/$CERT_MANAGER_VERSION/cert-manager.yaml
kubectl wait --for=condition=available --timeout=600s deployment/cert-manager-webhook -n cert-manager

# Install KServe + Runtimes
kubectl apply -f https://github.com/kserve/kserve/releases/download/$KSERVE_VERSION/kserve.yaml
kubectl wait --for=condition=ready pod -l control-plane=kserve-controller-manager -n kserve --timeout=300s
kubectl apply -f https://github.com/kserve/kserve/releases/download/$KSERVE_VERSION/kserve-runtimes.yaml
```

## Tests
Based on https://kserve.github.io/website/0.10/get_started/first_isvc/#5-perform-inference

```bash
kubectl create namespace kserve-test

kubectl apply -n kserve-test -f - <<EOF
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model"
EOF
```

```bash
# Url from Inference Service
kubectl get inferenceservices sklearn-iris -n kserve-test

NAME           URL                                                    READY   PREV   LATEST   PREVROLLEDOUTREVISION   LATESTREADYREVISION                    AGE
sklearn-iris   http://sklearn-iris.kserve-test.10.89.0.200.sslip.io   True           100                              sklearn-iris-predictor-default-00001   5m38s

curl http://sklearn-iris.kserve-test.10.89.0.200.sslip.io/v1/models/sklearn-iris:predict -d @../kserve/samples/input-iris.json
{"predictions":[1,1]}% 

# Url from Knative Service
k get ksvc -n kserve-test
NAME                             URL                                                                      LATESTCREATED                          LATESTREADY                            READY   REASON
sklearn-iris-predictor-default   http://sklearn-iris-predictor-default.kserve-test.10.89.0.200.sslip.io   sklearn-iris-predictor-default-00001   sklearn-iris-predictor-default-00001   True

curl http://sklearn-iris-predictor-default.kserve-test.10.89.0.200.sslip.io/v1/models/sklearn-iris:predict -d @../kserve/samples/input-iris.json
{"predictions":[1,1]}%
```
