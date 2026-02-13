# Non-operator MaaS on OpenShift v4.20.12 with Red Hat AI v3.2.0

## Prerequisites

```bash
# Install Red Hat build of Leader Worker Set v1.0.0
oc apply -f 01-install_leaderworkerset_api.yaml
oc apply -f 02-set_up_the_lws_api.yaml

# Check if you have lws-controller-manager & openshift-lws-operator pods
oc get deployments --namespace openshift-lws-operator

# Create GatewayClass, this will also install Red Hat OpenShift
# Service Mesh 3 v3.1.0
oc apply -f 03-install_gateway_api_controller.yaml
oc get gatewayclass openshift-default

# Install & configure Red Hat Connectivity Link v1.2.1,
# Authorino Operator v1.2.4, DNS Operator v1.2.0, Limitador v1.2.0
oc apply -f 04-install_rhcl.yaml
oc apply -f 05-create_a_connect_link_instance.yaml
oc get deployments -n kuadrant-system
oc apply -f 06-set_up_the_inference_gateway.yaml

# Install & configure Red HAt OpenShift AI v3.2.0 and the Data
# Science Cluster
oc apply -f 07-install_rhoai3.yaml
oc apply -f 08-create_dsc.yaml
oc get pods -n redhat-ods-applications

# Create a maas-default-gateway
CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}')
oc apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: maas-default-gateway
  namespace: openshift-ingress
spec:
  gatewayClassName: openshift-default
  listeners:
   - name: http
     hostname: maas.${CLUSTER_DOMAIN}
     port: 80
     protocol: HTTP
     allowedRoutes:
       namespaces:
         from: All
EOF
oc wait --for=condition=Programmed gateway/maas-default-gateway -n openshift-ingress --timeout=60s
oc get gateway maas-default-gateway -n openshift-ingress

# Patch the OdhDashboardConfig
oc patch OdhDashboardConfig odh-dashboard-config -n redhat-ods-applications --type=merge --patch "
spec:
  dashboardConfig:
    disableModelRegistry: false
    disableModelCatalog: false
    disableKServeMetrics: false
    genAiStudio: true
    modelAsService: true
    disableLMEval: false"
```

## MaaS

```bash
# Deploy MaaS objects
oc apply -f 09-create_maas_objects.yaml
oc apply -f 10-authpolicy_for_maas.yaml

# Patch the MaaS API AuthPolicy with your cluster's audience
AUD="$(oc create token default --duration=10m 2>/dev/null | cut -d. -f2 | jq -Rr '@base64d | fromjson | .aud[0]' 2>/dev/null)"
oc patch authpolicy gateway-auth-policy -n openshift-ingress --type=merge --patch "
spec:
  rules:
    authentication:
      openshift-identities:
        kubernetesTokenReview:
          audiences:
            - ${AUD}
            - maas-default-gateway-sa"

# Install rate limiting policies and tier to group mapping:
oc apply -f 11-token_rate_limit_policy_and_tier_to_grp.yaml
```

## Playing with LLMs and rate limiting

```bash
# Deploy Meta's OPT 125m CPU testing model simulator (does not require GPU)
oc apply -f 12-opt125m-cpu-model.yaml
oc get LLMInferenceService simulated -n llm

# Modify the LLMInferenceService to use the newly created maas-default-gateway
oc patch llminferenceservice simulated -n llm --type='json' -p='[
  {
    "op": "add",
    "path": "/spec/gateway/refs/-",
    "value": {
      "name": "maas-default-gateway",
      "namespace": "openshift-ingress"
    }
  }
]'

# Get Gateway endpoint
CLUSTER_DOMAIN=$(oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}') && \
HOST="http://maas.${CLUSTER_DOMAIN}" && \
echo "Gateway endpoint: $HOST"

# Get authentication endpoint
TOKEN_RESPONSE=$(curl -sSk \
  -H "Authorization: Bearer $(oc whoami -t)" \
  -H "Content-Type: application/json" \
  -X POST \
  -d '{"expiration": "10m"}' \
  "${HOST}/maas-api/v1/tokens") && \
TOKEN=$(echo $TOKEN_RESPONSE | jq -r .token) && \
echo "Token obtained: ${TOKEN:0:20}..."

# List available models
MODELS=$(curl -sSk ${HOST}/maas-api/v1/models \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $TOKEN" | jq -r .) && \
echo $MODELS | jq . && \
MODEL_NAME=$(echo $MODELS | jq -r '.data[0].id') && \
MODEL_URL=$(echo $MODELS | jq -r '.data[0].url') && \
echo "Model URL: $MODEL_URL"

# Test Model inference endpoint
curl -sSk -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"model\": \"${MODEL_NAME}\", \"prompt\": \"Hello\", \"max_tokens\": 50}" \
  "${MODEL_URL}/v1/completions" | jq

# Test Authorization Enforcement
curl -sSk -H "Content-Type: application/json" \
  -d "{\"model\": \"${MODEL_NAME}\", \"prompt\": \"Hello\", \"max_tokens\": 50}" \
  "${MODEL_URL}/v1/completions" -v

# Test rate limiting
for i in {1..16}; do
  curl -sSk -o /dev/null -w "%{http_code}\n" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"model\": \"${MODEL_NAME}\", \"prompt\": \"Hello\", \"max_tokens\": 50}" \
    "${MODEL_URL}/v1/completions"
done
```

## Verification

```bash
# Check all namespaces
oc get ns | grep -E "kuadrant-system|kserve|redhat-ods-applications|maas-api|llm"

# Check Gateway status
oc get gateway -n openshift-ingress maas-default-gateway

# Check policies
oc get authpolicy -A
oc get tokenratelimitpolicy -A
oc get ratelimitpolicy -A

# Check MaaS API
oc get pods -n maas-api -l app.kubernetes.io/name=maas-api
oc get svc -n maas-api
# Test connectivity between Authorino and MaaS
AUTHORINO_POD=$(oc get pods -n kuadrant-system -l authorino-resource=authorino -o jsonpath='{.items[0].metadata.name}')
oc exec -n kuadrant-system $AUTHORINO_POD -- curl -s http://maas-api.maas-api.svc.cluster.local:8080/health

# Check Kuadrant operators
oc get pods -n kuadrant-system

# Check RHOAI/KServe
oc get pods -n kserve
oc get pods -n redhat-ods-applications

# Check pods
oc get pods -n redhat-ods-applications && \
oc get pods -n kuadrant-system && \
oc get pods -n kserve

# Check Gateway status
oc get gateway -n openshift-ingress maas-default-gateway

# Check policies are enforced
oc get authpolicy -A && \
oc get tokenratelimitpolicy -A
```

---

_Last update: Fri 13 Feb 2026 01:22:55 UTC_
