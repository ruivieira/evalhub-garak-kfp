# EvalHub + Garak on OpenDataHub (KFP)

End-to-end guide for running Garak red-teaming scans through EvalHub on an OpenDataHub (ODH) cluster.
Garak scans run as Kubeflow Pipelines (KFP / DSP v2) jobs orchestrated through EvalHub's evaluation API.

## Architecture

```
opendatahub namespace          tenant namespace
┌─────────────────────┐        ┌────────────────────────────────┐
│  TrustyAI operator  │        │  DSPA (KFP backend)            │
│  EvalHub service    │──────▶ │  Garak KFP adapter pod (job)   │
│  garak-kfp provider │        │  MinIO (artifact store)        │
│  (built-in, OOTB)   │        └────────────────────────────────┘
└─────────────────────┘
```

EvalHub itself runs in the operator namespace (`opendatahub`). Jobs and DSPA run in a separate tenant namespace. The `garak-kfp` provider is shipped by the TrustyAI operator — no manual registration needed.

The tenant namespace is specified at job submission time via the `X-Tenant` header. Multiple tenant namespaces are supported; repeat steps 2–6 for each one.

## Prerequisites

- OpenShift cluster with OpenDataHub operator installed
- `oc` CLI logged in with cluster-admin access
- A model inference endpoint (vLLM, OpenAI-compatible)

---

## 1. Configure the DataScienceCluster

Enable TrustyAI (which includes the EvalHub operator) and Data Science Pipelines in your `DataScienceCluster`. Edit your existing DSC or apply a new one:

```yaml
apiVersion: datasciencecluster.opendatahub.io/v1
kind: DataScienceCluster
metadata:
  name: default
spec:
  components:
    trustyai:
      managementState: Managed
    datasciencepipelines:
      managementState: Managed
    dashboard:
      managementState: Managed
```

```bash
oc apply -f datasciencecluster.yaml
```

Wait for the operator to become ready:

```bash
oc rollout status deployment/trustyai-service-operator-controller-manager \
  -n opendatahub --timeout=120s
```

---

## 2. Deploy EvalHub

The `garak-kfp` provider is shipped by the operator and available automatically. The EvalHub CR just needs to list it:

```bash
oc apply -f resources/evalhub-cr.yaml -n opendatahub
oc rollout status deployment/evalhub -n opendatahub --timeout=180s
```

Expose the route if one does not already exist:

```bash
oc get route evalhub -n opendatahub &>/dev/null || \
  oc expose svc/evalhub -n opendatahub --name=evalhub
```

Set the EvalHub URL:

```bash
export EVALHUB_URL="https://$(oc get route evalhub -n opendatahub -o jsonpath='{.spec.host}')"
```

---

## 3. Create and Label the Tenant Namespace

The operator watches for namespaces labelled with `evalhub.trustyai.opendatahub.io/tenant` and automatically provisions all required job RBAC (service account, role bindings, service CA ConfigMap) in that namespace.

```bash
export NS=test

oc new-project "$NS"
oc label namespace "$NS" evalhub.trustyai.opendatahub.io/tenant=true
```

Wait for the operator to reconcile — it will create the job service account and bindings in `$NS`:

```bash
oc get serviceaccount evalhub-opendatahub-job -n "$NS" --timeout=60s
```

> To onboard additional tenant namespaces, create and label them the same way. Each labelled namespace is independently managed.

### Create a tenant user and grant API access

EvalHub enforces access via Kubernetes SAR checks. Create a ServiceAccount to act as the tenant API user, and bind it to the required evaluation permissions in the tenant namespace:

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: evalhub-user
  namespace: $NS
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: evalhub-evaluator
  namespace: $NS
rules:
  - apiGroups: [trustyai.opendatahub.io]
    resources: [evaluations, collections, providers]
    verbs: [get, list, create, update, delete]
  - apiGroups: [mlflow.kubeflow.org]
    resources: [experiments]
    verbs: [create, get]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: evalhub-evaluator-binding
  namespace: $NS
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: evalhub-evaluator
subjects:
  - kind: ServiceAccount
    name: evalhub-user
    namespace: $NS
EOF
```

Set the remaining environment variables:

```bash
export TOKEN=$(oc create token evalhub-user -n "$NS" --duration=8h)
export MODEL_URL="http://vllm-server.${NS}.svc.cluster.local:8000"
export MODEL_NAME="tinyllama"
```

Verify EvalHub is healthy and the provider is available:

```bash
curl -sk -H "Authorization: Bearer $TOKEN" "$EVALHUB_URL/api/v1/health"
curl -sk -H "Authorization: Bearer $TOKEN" \
  -H "X-Tenant: $NS" \
  "$EVALHUB_URL/api/v1/evaluations/providers" | jq '[.items[].resource.id]'
```

---

## 4. Deploy Data Science Pipelines (DSPA)

Deploy a `DataSciencePipelinesApplication` in the tenant namespace. This provisions the KFP API server, scheduler, and a built-in MinIO artifact store.

```bash
cat <<EOF | oc apply -f -
apiVersion: datasciencepipelinesapplications.opendatahub.io/v1alpha1
kind: DataSciencePipelinesApplication
metadata:
  name: dspa
  namespace: $NS
spec:
  dspVersion: v2
  objectStorage:
    disableHealthCheck: false
    enableExternalRoute: false
    minio:
      deploy: true
      image: quay.io/opendatahub/minio:RELEASE.2019-08-14T20-37-41Z-license-compliance
EOF
```

Wait for the DSP components:

```bash
oc rollout status deployment/ds-pipeline-dspa -n "$NS" --timeout=300s
oc rollout status deployment/ds-pipeline-scheduledworkflow-dspa -n "$NS" --timeout=120s
```

---

## 5. Grant DSP Access to the Job Service Account

The Garak KFP adapter pod needs permission to interact with the DSP API. This is specific to KFP and is not provisioned by the operator automatically.

```bash
JOB_SA="evalhub-opendatahub-job"   # format: evalhub-<operator-namespace>-job

cat <<EOF | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: evalhub-jobs-dspa-api
  namespace: $NS
rules:
- apiGroups: ["datasciencepipelinesapplications.opendatahub.io"]
  resources: ["datasciencepipelinesapplications/api"]
  verbs: ["get", "create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: evalhub-jobs-dspa-api
  namespace: $NS
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: evalhub-jobs-dspa-api
subjects:
- kind: ServiceAccount
  name: $JOB_SA
  namespace: $NS
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: evalhub-jobs-pipeline-management
  namespace: $NS
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ds-pipeline-dspa
subjects:
- kind: ServiceAccount
  name: $JOB_SA
  namespace: $NS
EOF
```

---

## 6. Patch the MinIO Secret

The DSP MinIO secret uses ODH-style keys (`accesskey`, `secretkey`). The Garak KFP pipeline injects the secret into pipeline pods via `kubernetes.use_secret_as_env` and expects AWS-style keys. Patch the secret once (idempotent):

```bash
ACCESS=$(oc get secret ds-pipeline-s3-dspa -n "$NS" \
  -o jsonpath='{.data.accesskey}' | base64 -d)
SECRET=$(oc get secret ds-pipeline-s3-dspa -n "$NS" \
  -o jsonpath='{.data.secretkey}' | base64 -d)
ENDPOINT="http://minio-dspa.${NS}.svc.cluster.local:9000"

oc patch secret ds-pipeline-s3-dspa -n "$NS" --type=merge -p "{
  \"stringData\": {
    \"AWS_ACCESS_KEY_ID\":     \"$ACCESS\",
    \"AWS_SECRET_ACCESS_KEY\": \"$SECRET\",
    \"AWS_S3_ENDPOINT\":       \"$ENDPOINT\",
    \"AWS_S3_BUCKET\":         \"mlpipeline\",
    \"AWS_DEFAULT_REGION\":    \"us-east-1\"
  }
}"
```

> The internal cluster endpoint is used deliberately. The DSP CA bundle (mounted as `dsp-ca.crt` in pipeline pods) does not cover the external route certificate, so boto3 would fail SSL verification through the external route.

---

## 7. Submit a Garak Scan

The `X-Tenant` header tells EvalHub which namespace to run the job in. Set it to the tenant namespace you labelled in step 3.

### Standard benchmark (e.g. `quick`)

```bash
KFP_EP="https://ds-pipeline-dspa.${NS}.svc.cluster.local:8443"

curl -sk -X POST "$EVALHUB_URL/api/v1/evaluations/jobs" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-Tenant: $NS" \
  -d "{
    \"model\": {
      \"url\": \"$MODEL_URL\",
      \"name\": \"$MODEL_NAME\"
    },
    \"benchmarks\": [{
      \"id\": \"quick\",
      \"provider_id\": \"garak-kfp\",
      \"parameters\": {
        \"kfp_config\": {
          \"endpoint\":       \"$KFP_EP\",
          \"namespace\":      \"$NS\",
          \"s3_secret_name\": \"ds-pipeline-s3-dspa\",
          \"s3_endpoint\":    \"http://minio-dspa.${NS}.svc.cluster.local:9000\",
          \"s3_bucket\":      \"mlpipeline\",
          \"verify_ssl\":     false
        }
      }
    }],
    \"timeout_minutes\": 60,
    \"retry_attempts\": 1
  }"
```

### Intents benchmark (ART Intents — requires a judge model)

The `intents` benchmark runs the ART Intents red-teaming framework as a KFP pipeline. It requires a judge model and, optionally, an SDG model for synthetic dataset generation.

> **Requires `jq`** to build the payload.

```bash
KFP_EP="https://ds-pipeline-dspa.${NS}.svc.cluster.local:8443"

# Model endpoint for judge/attacker (append /v1 for OpenAI-compatible APIs)
INTENTS_URL="${MODEL_URL}/v1"
INTENTS_NAME="$MODEL_NAME"

# SDG model — litellm requires a provider prefix (e.g. openai/model_name)
SDG_URL="${MODEL_URL}/v1"
SDG_NAME="openai/${MODEL_NAME}"
SDG_FLOW="major-sage-742"          # Your SDG flow ID

curl -sk -X POST "$EVALHUB_URL/api/v1/evaluations/jobs" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "X-Tenant: $NS" \
  -d "$(jq -n \
    --arg model_url  "$MODEL_URL" \
    --arg model_name "$MODEL_NAME" \
    --arg kfp_ep     "$KFP_EP" \
    --arg kfp_ns     "$NS" \
    --arg judge_url  "$INTENTS_URL" \
    --arg judge_name "$INTENTS_NAME" \
    --arg sdg_url    "$SDG_URL" \
    --arg sdg_name   "$SDG_NAME" \
    --arg sdg_flow   "$SDG_FLOW" \
    '{
      model: { url: $model_url, name: $model_name },
      benchmarks: [{
        id: "intents",
        provider_id: "garak-kfp",
        parameters: {
          art_intents: true,
          intents_models: {
            judge: { url: $judge_url, name: $judge_name },
            sdg:   { url: $sdg_url,   name: $sdg_name }
          },
          sdg_model:    $sdg_url,
          sdg_api_base: $sdg_url,
          sdg_flow_id:  $sdg_flow,
          kfp_config: {
            endpoint:       $kfp_ep,
            namespace:      $kfp_ns,
            s3_secret_name: "ds-pipeline-s3-dspa",
            s3_endpoint:    ("http://minio-dspa." + $kfp_ns + ".svc.cluster.local:9000"),
            s3_bucket:      "mlpipeline",
            verify_ssl:     false
          }
        }
      }],
      timeout_minutes: 720,
      retry_attempts: 1
    }')"
```

---

## 8. Check Job Status

```bash
JOB_ID="<id from submission response>"

curl -sk -H "Authorization: Bearer $TOKEN" \
  "$EVALHUB_URL/api/v1/evaluations/jobs/$JOB_ID" | jq '{
    id:      .resource.id,
    state:   .status.state,
    message: .status.message.message
  }'
```

Track the KFP pipeline run directly:

```bash
KFP_TOKEN=$(oc create token evalhub-opendatahub-job -n "$NS" --duration=1h)
curl -sk -H "Authorization: Bearer $KFP_TOKEN" \
  "${KFP_EP}/apis/v2beta1/runs" | jq '[.runs[] | {name: .display_name, state: .state}]'
```

---

## Troubleshooting

### EvalHub pod not ready

```bash
oc describe deployment evalhub -n opendatahub
oc logs deployment/evalhub -n opendatahub -c evalhub
oc logs deployment/evalhub -n opendatahub -c kube-rbac-proxy
```

### Job SA not created in tenant namespace

Confirm the namespace has the tenant label; the operator will reconcile within seconds:

```bash
oc get namespace "$NS" --show-labels
# Expected: evalhub.trustyai.opendatahub.io/tenant=true

oc get serviceaccount evalhub-opendatahub-job -n "$NS"
```

### KFP pipeline run fails immediately

Verify the MinIO secret patch (step 6) and DSP role binding (step 5):

```bash
oc get secret ds-pipeline-s3-dspa -n "$NS" \
  -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d && echo
oc get rolebinding evalhub-jobs-pipeline-management -n "$NS"
```

### 403 Forbidden on API calls

Verify the tenant SA has the evaluator Role in the tenant namespace:

```bash
oc auth can-i create evaluations.trustyai.opendatahub.io \
  --as="system:serviceaccount:${NS}:evalhub-user" -n "$NS"
```

### Job pod cannot reach EvalHub

```bash
oc auth can-i create status-events \
  --as="system:serviceaccount:${NS}:evalhub-opendatahub-job" -n "$NS"

oc get cm evalhub-service-ca -n "$NS" \
  -o jsonpath='{.data.service-ca\.crt}' | head -3
```

### Provider not listed

Confirm the EvalHub CR includes `garak-kfp` and the operator has reconciled:

```bash
oc get evalhub evalhub -n opendatahub -o jsonpath='{.spec.providers}'
oc get cm -n opendatahub -l trustyai.opendatahub.io/evalhub-provider-name=garak-kfp
```

---

## Cleanup

```bash
# Remove EvalHub (operator cleans up owned resources via finaliser)
oc delete evalhub evalhub -n opendatahub

# Remove DSPA
oc delete dspa dspa -n "$NS"

# Remove tenant namespace (operator cleans up job resources when label is removed or namespace deleted)
oc delete project "$NS"
```

---

## License

Apache License 2.0. See [LICENSE](LICENSE).
