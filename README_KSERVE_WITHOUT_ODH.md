# KServe Setup without ODH

```bash
# Install KServe
oc apply -f kserve/kserve.yaml
oc wait --for=condition=ready pod -l control-plane=kserve-controller-manager -n kserve --timeout=300s

# Patch KServe config
# 1) to override default images because of user-permission issues in OCP
oc apply -f kserve/kserve-config-patch-istio.yaml

# Restart the kserve controller
oc rollout restart deployment kserve-controller-manager -n kserve

# Install KServe built-in serving runtimes
oc wait --for=condition=ready pod -l control-plane=kserve-controller-manager -n kserve --timeout=300s
oc apply -f kserve/kserve-runtimes.yaml

# Add NetworkPolicies to allow traffic to kserve webhook
oc apply -f kserve/networkpolicies.yaml
```