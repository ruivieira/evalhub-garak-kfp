# EvalHub + Garak on OpenDataHub

Use the **`:latest`** images for both the TrustyAI operator and EvalHub (e.g. `quay.io/trustyai/trustyai-service-operator:latest` and `quay.io/trustyai/eval-hub:latest`).

End-to-end guide for deploying EvalHub with Garak red-teaming on an OpenDataHub (ODH) cluster. Garak scans run as Kubeflow Pipelines (KFP / DSP v2) jobs, orchestrated through EvalHub's evaluation API.

## Prerequisites

- OpenShift cluster with OpenDataHub installed
- `oc` CLI with cluster-admin access
- A model inference endpoint (vLLM, OpenAI-compatible, etc.)
## 1. Configure the DataScienceCluster

Enable TrustyAI (which includes the EvalHub operator) and Data Science Pipelines in your `DataScienceCluster`:

```yaml
apiVersion: datasciencecluster.opendatahub.io/v1
kind: DataScienceCluster
metadata:
  name: default
spec:
  components:
    trustyai:
      managementState: Managed
      eval:
        lmeval:
          permitOnline: allow
          permitCodeExecution: allow
    datasciencepipelines:
      managementState: Managed
    kserve:
      managementState: Managed
      serving:
        ingressGateway:
          certificate:
            type: SelfSigned
        managementState: Managed
        name: knative-serving
    dashboard:
      managementState: Managed
```

Apply:

```bash
oc apply -f datasciencecluster.yaml
```

## 2. Create the Namespace

```bash
export NS=evalhub-garak
oc new-project "$NS"
```

## 3. Register provider ConfigMaps

Register the Garak (non-KfP) and Garak (KfP) providers so the TrustyAI operator can expose them to EvalHub. These ConfigMaps must be applied in the **operator namespace** (e.g. `opendatahub`).

**Garak (non-KfP)** — runs Garak directly in a Kubernetes Job:

```bash
oc apply -f resources/evalhub-provider-garak.yaml -n opendatahub
```

**Garak (KfP)** — runs the KFP adapter in a Job; the adapter submits a Garak pipeline to Data Science Pipelines:

```bash
oc apply -f resources/evalhub-provider-garak-kfp.yaml -n opendatahub
```

If your operator runs in a different namespace, replace `opendatahub` with that namespace. The EvalHub CR in the next step lists both providers (`dev-garak` and `dev-garak-kfp`); the operator copies these ConfigMaps into the EvalHub instance namespace when you deploy it.

> **Why `dev-garak` instead of `garak`?** The TrustyAI operator ships built-in provider definitions named `garak` and `garak-kfp`. Using distinct names avoids collisions and ensures your custom ConfigMaps are the ones picked up.

## 4. Deploy EvalHub

> **Database:** PostgreSQL is supported and recommended for production. For instruction simplicity, this guide uses the bundled database (SQLite).

```yaml
# See resources/evalhub-cr.yaml (namespace set at apply time via -n)
apiVersion: trustyai.opendatahub.io/v1alpha1
kind: EvalHub
metadata:
  name: evalhub
spec:
  replicas: 1
  providers:
    - dev-garak
    - dev-garak-kfp
```

Apply:

```bash
oc apply -f resources/evalhub-cr.yaml -n "$NS"
```

## 5. Verify providers

Check that EvalHub sees both Garak providers and their benchmarks. Expose the service with a route (if needed), set the URL and token, then call the API:

```bash
# Create a route if one does not exist
oc get route evalhub -n "$NS" &>/dev/null || oc expose svc/evalhub -n "$NS" --name=evalhub

# Set EvalHub URL and token for later steps
export EVALHUB_URL="https://$(oc get route evalhub -n "$NS" -o jsonpath='{.spec.host}')"
export TOKEN=$(oc create token evalhub-jobs -n "$NS" --duration=1h)
echo "EVALHUB_URL=$EVALHUB_URL"

# Model endpoint (example only — change to match your deployment)
export MODEL_URL="http://vllm-server.test.svc.cluster.local:8000"
export MODEL_NAME="tinyllama"
```

```bash
# Health
curl -sk -H "Authorization: Bearer $TOKEN" "$EVALHUB_URL/api/v1/health"
```

Expected:

```json
{"status":"healthy","timestamp":"2026-03-02T00:08:36.023114392Z"}
```

```bash
# List providers (includes benchmarks; should include dev-garak and dev-garak-kfp)
curl -sk -H "Authorization: Bearer $TOKEN" "$EVALHUB_URL/api/v1/evaluations/providers" | jq .
```

Expected (sample; you should see both `dev-garak-kfp` and `dev-garak` with their benchmarks):

```json
{
  "limit": 0,
  "total_count": 2,
  "items": [
    {
      "resource": { "id": "dev-garak-kfp", "read_only": true, "owner": "system" },
      "name": "Garak KFP (dev)",
      "description": "LLM vulnerability scanner delegating execution to a Kubeflow Pipeline",
      "benchmarks": [
        { "id": "quick", "name": "Quick Scan", "category": "safety", "metrics": ["attack_success_rate"] },
        { "id": "owasp_llm_top10", "name": "OWASP LLM Top 10", "category": "security", "metrics": ["attack_success_rate"] }
      ]
    },
    {
      "resource": { "id": "dev-garak", "read_only": true, "owner": "system" },
      "name": "Garak (dev)",
      "description": "LLM vulnerability scanner and red-teaming framework",
      "benchmarks": [
        { "id": "toxicity", "name": "Toxicity Detection", "category": "safety", "metrics": ["toxicity_rate", "severity_score"] },
        { "id": "prompt_injection", "name": "Prompt Injection", "category": "security", "metrics": ["injection_success_rate", "defense_effectiveness"] }
      ]
    }
  ]
}
```

## 6. Run a Garak (non-KfP) Scan

Runs Garak directly in a Kubernetes Job. Use `provider_id: "dev-garak"`. No DSP required. Use `EVALHUB_URL` and `TOKEN` from step 5.

Available benchmark IDs for the `dev-garak` provider: `quick`, `toxicity`, `bias_detection`, `pii_leakage`, `prompt_injection`.

```bash
# Submit (example: quick benchmark)
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$EVALHUB_URL/api/v1/evaluations/jobs" \
  -d "{
    \"model\": {
      \"url\": \"$MODEL_URL\",
      \"name\": \"$MODEL_NAME\"
    },
    \"benchmarks\": [
      {
        \"id\": \"quick\",
        \"provider_id\": \"dev-garak\",
        \"parameters\": {}
      }
    ]
  }"

```

Expected:

```json
{
  "resource": {
    "id": "c78100bf-c3ef-4fb5-a01e-fa268178cea3",
    "tenant": "",
    "created_at": "2026-03-02T00:25:58.390295945Z"
  },
  "status": {
    "state": "pending",
    "message": {
      "message": "Evaluation job created",
      "message_code": "evaluation_job_created"
    }
  },
  "model": {
    "url": "...",
    "name": "..."
  },
  "benchmarks": [
    { "id": "quick", "provider_id": "dev-garak" }
  ]
}
```

```bash
# Check job status (use job id from response)
JOB_ID="c78100bf-c3ef-4fb5-a01e-fa268178cea3"
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$EVALHUB_URL/api/v1/evaluations/jobs/$JOB_ID" | jq .
```

## 7. Install Data Science Pipelines (KFP)

Data Science Pipelines provides the KFP backend for running Garak (KfP) scans.

```bash
oc apply -f - <<EOF
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
    externalStorage:
      basePath: ""
      bucket: ""
      host: ""
      port: ""
      region: us-east-1
      s3CredentialsSecret:
        accessKey: AWS_ACCESS_KEY_ID
        secretKey: AWS_SECRET_ACCESS_KEY
        secretName: aws-connection-s3
      scheme: https
EOF
```

Wait for DSP pods:

```bash
oc get pods -n "$NS" | grep -E "dspa|ds-pipeline"
export KFP_ENDPOINT="https://$(oc get routes ds-pipeline-dspa -n "$NS" -o jsonpath='{.spec.host}')"
echo "$KFP_ENDPOINT"
```

## 8. Run a Garak (KfP) Scan

Runs the Garak scan as a Kubeflow Pipeline. Use `provider_id: "dev-garak-kfp"`. Requires Data Science Pipelines (step 7).

```bash
# Submit (example: quick benchmark)
curl -sk -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$EVALHUB_URL/api/v1/evaluations/jobs" \
  -d "{
    \"model\": {
      \"url\": \"$MODEL_URL\",
      \"name\": \"$MODEL_NAME\"
    },
    \"benchmarks\": [
      {
        \"id\": \"quick\",
        \"provider_id\": \"dev-garak-kfp\",
        \"parameters\": {}
      }
    ],
    "experiment": {
      "name": "garak-red-team-test",
      "tags": { "env": "dev" }
    },
    "timeout_minutes": 60
  }'

# Check job status
JOB_ID="<job-id-from-response>"
curl -sk -H "Authorization: Bearer $TOKEN" \
  "$EVALHUB_URL/api/v1/evaluations/jobs/$JOB_ID" | jq .
```

## Predefined Garak Benchmarks

| Benchmark ID | Description |
|---|---|
| `quick` | Fast smoke test across common probes |
| `owasp_llm_top10` | OWASP LLM Top 10 compliance |

## Troubleshooting

### EvalHub pod not ready

```bash
oc describe deployment evalhub -n "$NS"
oc logs -n "$NS" deployment/evalhub -c evalhub
oc logs -n "$NS" deployment/evalhub -c kube-rbac-proxy
```

### KFP run fails with authorisation errors

- Check the DSP role binding:

```bash
oc get rolebinding -n "$NS" | grep ds-pipeline
```

### Job pods fail to authenticate with EvalHub

```bash
# Check RBAC
oc auth can-i get evalhubs/proxy \
  --as=system:serviceaccount:$NS:evalhub-jobs -n "$NS"

# Check service CA injection
oc get cm evalhub-service-ca -n "$NS" -o jsonpath='{.data.service-ca\.crt}' | head -5
```

## Cleanup

```bash
# Delete EvalHub (operator cleans up owned resources via finaliser)
oc delete evalhub evalhub -n "$NS"

# Delete DSP
oc delete dspa dspa -n "$NS"

# Delete namespace
oc delete project "$NS"
```

## License

Apache License 2.0. See [LICENSE](LICENSE).
