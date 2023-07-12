# OpenShift + KServe + Knative with Kourier

## Prerequisites
* An OpenShift cluster, configured with a reachable domain for OCP routes
* OC client pointing to that cluster


## Installation with Kourier
```bash
# Install OpenShift Serverless operator
oc apply -f serverless/operator.yaml
oc wait --for=condition=ready pod -l name=knative-openshift -n openshift-serverless --timeout=300s
oc wait --for=condition=ready pod -l name=knative-openshift-ingress -n openshift-serverless --timeout=300s
oc wait --for=condition=ready pod -l name=knative-operator -n openshift-serverless --timeout=300s

# Create an Knative instance
oc apply -f serverless/knativeserving-kourier.yaml
oc wait --for=condition=ready pod -l app=controller -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=autoscaler-hpa -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=domain-mapping -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=webhook -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=activator -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=autoscaler -n knative-serving --timeout=300s
oc wait --for=condition=ready pod -l app=net-kourier-controller -n knative-serving-ingress --timeout=300s
oc wait --for=condition=ready pod -l app=3scale-kourier-gateway -n knative-serving-ingress --timeout=300s

# Install cert-manager operator
oc apply -f cert-manager/operator.yaml
oc wait --for=condition=ready pod -l app=webhook -n cert-manager --timeout=300s
oc wait --for=condition=ready pod -l app=cainjector -n cert-manager --timeout=300s
oc wait --for=condition=ready pod -l app=cert-manager -n cert-manager --timeout=300s

# Install KServe
oc apply -f kserve/kserve.yaml
oc wait --for=condition=ready pod -l control-plane=kserve-controller-manager -n kserve --timeout=300s

# Patch KServe config
# 1) to enable Kourier according to https://kserve.github.io/website/0.10/admin/serverless/kourier_networking/#install-kourier-networking-layer
# 2) to override default images because of user-permission issues in OCP
oc apply -f kserve/kserve-config-patch-kourier.yaml

# Restart the kserve controller
oc rollout restart deployment kserve-controller-manager -n kserve

# Install KServe built-in serving runtimes
oc wait --for=condition=ready pod -l control-plane=kserve-controller-manager -n kserve --timeout=300s
oc apply -f kserve/kserve-runtimes.yaml
```