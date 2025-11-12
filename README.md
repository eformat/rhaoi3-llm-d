# Qwen3-0.6B LLM Deployment

This folder contains Kustomize files to deploy llm-d with the Qwen3-0.6B model on OpenShift, including Prometheus and Grafana dashboard integration.

## Prerequisites

- OpenShift cluster with GPU nodes
- OpenShift AI (RHOAI) operator installed
- `oc` CLI tool installed and configured
- `kubectl` with kustomize support (or standalone `kustomize`)

## Deployment

1. **Login to your OpenShift cluster:**
   ```bash
   oc login --server=<your-cluster-url>
   ```

2. **Deploy from the llm-d directory (recommended - includes base namespace setup):**
   ```bash
   cd llm-d
   kubectl apply -k .
   ```

   Or from the repository root:
   ```bash
   kubectl apply -k llm-d
   ```

3. **Wait for the deployment to be ready:**
   ```bash
   oc wait --for=condition=Ready llminferenceservice/qwen -n demo-llm --timeout=600s
   ```

   Or watch pod status:
   ```bash
   oc get pods -n demo-llm -w
   ```

4. **Get the service URL:**
   ```bash
   export LLM_URL=$(oc get llminferenceservice qwen -n demo-llm -o jsonpath='{.status.url}')
   echo "LLM Service URL: $LLM_URL"
   ```

## Testing

### Set up test variables

```bash

# Define test prompts
export SHORT_TEXT="What is the capital of France?"

export LONG_TEXT_200_WORDS="Artificial intelligence has become one of the most transformative technologies of the 21st century, reshaping industries from healthcare to finance, transportation to entertainment. Machine learning algorithms can now analyze vast amounts of data, recognize complex patterns, and make predictions with remarkable accuracy. Deep learning networks have achieved superhuman performance in tasks like image recognition and natural language processing. However, the rapid advancement of AI also raises important ethical questions about privacy, bias, job displacement, and the concentration of power in the hands of a few large technology companies. As AI systems become more sophisticated and autonomous, society must grapple with how to ensure they align with human values and serve the common good. Researchers are exploring various approaches to make AI more transparent, fair, and accountable, including explainable AI, algorithmic auditing, and robust governance frameworks. The future of AI will depend not just on technical breakthroughs but also on thoughtful policy decisions and broad societal engagement with these critical issues. Education and public discourse will be essential to help people understand both the opportunities and risks that AI presents."
```

### Test with short prompt

```bash
curl -s $LLM_URL/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "prompt": "'"$SHORT_TEXT"'",
    "max_tokens": 50
  }' | jq
```

### Test with long prompt (200 words)

```bash
curl -s $LLM_URL/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "prompt": "'"$LONG_TEXT_200_WORDS"'",
    "max_tokens": 50
  }' | jq
```

### Test streaming completion

```bash
curl -s $LLM_URL/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "prompt": "'"$SHORT_TEXT"'",
    "max_tokens": 50,
    "stream": true
  }'
```

## Monitoring

### Deploy Monitoring Stack

The repository includes a complete Prometheus and Grafana monitoring stack with a pre-configured LLM performance dashboard.

1. **Deploy the monitoring stack:**
   ```bash
   kubectl apply -k monitoring
   ```

2. **Wait for Grafana to be ready:**
   ```bash
   oc wait --for=condition=ready pod -l app=grafana -n llm-d-monitoring --timeout=300s
   ```

3. **Get the Grafana URL:**
   ```bash
   export GRAFANA_URL=$(oc get route grafana-secure -n llm-d-monitoring -o jsonpath='{.spec.host}')
   echo "Grafana URL: https://$GRAFANA_URL"
   ```

4. **Access Grafana:**
   - Open the Grafana URL in your browser
   - Default credentials: `admin` / `admin` (you'll be prompted to change the password)
   - Navigate to Dashboards → LLM Performance Dashboard

### LLM Performance Dashboard

![Grafana Dashboard](assets/grafana.png)

### Monitoring Features

The monitoring stack includes:

- **Prometheus:** Collects metrics from the LLM inference service
- **Grafana:** Visualizes metrics with a pre-configured dashboard
- **LLM Performance Dashboard:** Shows request latency, throughput, token generation rates, and more
- **Metrics endpoint:** Available at the model server endpoint `/metrics`

## Configuration

The deployment is configured with:
- **Model:** Qwen3-0.6B from HuggingFace
- **Replicas:** 2 (for high availability)
- **GPU:** 1 NVIDIA GPU per replica
- **Memory:** 8Gi per replica
- **CPU:** 1 core per replica
- **Auth:** Disabled (set `security.opendatahub.io/enable-auth: 'true'` to enable)

## Troubleshooting

**Check pod status:**
```bash
oc get pods -n demo-llm -l serving.kserve.io/inferenceservice=qwen
```

**View logs:**
```bash
oc logs -n demo-llm -l serving.kserve.io/inferenceservice=qwen -c main --tail=100 -f
```

**Check service status:**
```bash
oc get llminferenceservice qwen -n demo-llm -o yaml
```

**Verify route URL:**
```bash
oc get llminferenceservice qwen -n demo-llm -o jsonpath='{.status.url}'
```

## Cleanup

To remove the deployment:

```bash
kubectl delete -k llm-d
```

Or from the llm-d directory:

```bash
cd llm-d
kubectl delete -k .
```
