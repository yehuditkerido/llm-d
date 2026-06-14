# Multi-Model Routing

## Overview

This guide deploys the **Inference Payload Processor (IPP)** to enable serving multiple LLMs and LoRA adapters behind a single Gateway endpoint. IPP extracts the model name from the request body and routes to the appropriate InferencePool based on header matching.

Use this guide when you need to:

* Serve multiple base models (e.g., Qwen for chatbots, DeepSeek for reasoning)
* Route LoRA adapter requests to their base model's InferencePool
* Provide a unified API endpoint following the OpenAI specification

For simpler single-model deployments, see the [Optimized Baseline](../optimized-baseline/README.md) guide instead.

## Prerequisites

* Have the [proper client tools installed on your local system](../../helpers/client-setup/README.md) to use this guide.
* Checkout llm-d repo:

  ```bash
  export branch="main" # branch, tag, or commit hash
  git clone https://github.com/llm-d/llm-d.git && cd llm-d && git checkout ${branch}
  ```

* Set the following environment variables:

  ```bash
  export REPO_ROOT=$(realpath $(git rev-parse --show-toplevel))
  source ${REPO_ROOT}/guides/env.sh
  export GUIDE_NAME="multi-model-routing"
  export NAMESPACE="llm-d-multi-model"
  ```

* Install the Gateway API Inference Extension CRDs:

  ```bash
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/${GAIE_VERSION}/v1-manifests.yaml
  ```

* Create a target namespace for the installation:

  ```bash
  kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
  ```

* [Create the `llm-d-hf-token` secret in your target namespace](../../helpers/hf-token.md) with a valid HuggingFace token.

* **Multiple InferencePools deployed**, each serving a different base model. Follow the [Optimized Baseline](../optimized-baseline/README.md) guide for each pool, or [Multi-Inference Pool Setup](../workload-autoscaling/README.multi-inference-pool.md) for adding pools to an existing deployment.

* A Kubernetes Gateway (e.g., Istio, GKE, AgentGateway) deployed in your cluster. See [Gateway Infrastructure](../../docs/infrastructure/gateway/README.md).

## Step 1: Deploy IPP

Clone the IPP repository and deploy using Helm:

```bash
# Clone the IPP repository
git clone https://github.com/llm-d/llm-d-inference-payload-processor.git /tmp/ipp

# Install IPP (for Istio)
helm install ipp /tmp/ipp/config/charts/payload-processor \
    --set provider.name=istio \
    --set inferenceGateway.name=inference-gateway \
    -n ${NAMESPACE}
```

> [!NOTE]
> For GKE, use `--set provider.name=gke` instead of `istio`.
> For standalone (no gateway), omit the `provider.name` flag.

Verify IPP is running:

```bash
kubectl get pods -n ${NAMESPACE} -l app=payload-processor
```

## Step 2: Create Model Mapping ConfigMaps

Create ConfigMaps that map model names (including LoRA adapters) to their base models. Each ConfigMap defines one base model and its adapters, and must have the label `inference.llm-d.ai/ipp-managed: "true"`:

```bash
# ConfigMap for Qwen base model and its LoRA adapters
kubectl apply -n ${NAMESPACE} -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: qwen-model-mapping
  labels:
    inference.llm-d.ai/ipp-managed: "true"
data:
  baseModel: "Qwen/Qwen3-32B"
  adapters: |
    - food-review-1
    - travel-assistant
EOF

# ConfigMap for DeepSeek base model and its LoRA adapters
kubectl apply -n ${NAMESPACE} -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: deepseek-model-mapping
  labels:
    inference.llm-d.ai/ipp-managed: "true"
data:
  baseModel: "deepseek/DeepSeek-r1"
  adapters: |
    - ski-resorts
    - movie-critique
EOF
```

> [!IMPORTANT]
> All model names (base models and LoRA adapters) must be globally unique across all InferencePools.

## Step 3: Configure HTTPRoutes

Create HTTPRoutes that match on the `X-Gateway-Base-Model-Name` header injected by IPP. Each route maps a base model name to the InferencePool serving that model:

> [!NOTE]
> Replace `qwen-pool` and `deepseek-pool` with the names of your existing InferencePools. The `value` in the header match must exactly match the `baseModel` value in the corresponding ConfigMap from Step 2.

```bash
kubectl apply -n ${NAMESPACE} -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: qwen-route
spec:
  parentRefs:
  - name: inference-gateway
  rules:
  - matches:
    - headers:
      - name: X-Gateway-Base-Model-Name
        value: Qwen/Qwen3-32B
    backendRefs:
    - group: inference.networking.k8s.io
      kind: InferencePool
      name: qwen-pool
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: deepseek-route
spec:
  parentRefs:
  - name: inference-gateway
  rules:
  - matches:
    - headers:
      - name: X-Gateway-Base-Model-Name
        value: deepseek/DeepSeek-r1
    backendRefs:
    - group: inference.networking.k8s.io
      kind: InferencePool
      name: deepseek-pool
EOF
```

## Step 4: Test the Deployment

Get the Gateway IP and send test requests:

```bash
export GATEWAY_IP=$(kubectl get gateway inference-gateway -n ${NAMESPACE} -o jsonpath='{.status.addresses[0].value}')

# Request to Qwen base model
curl -X POST "http://${GATEWAY_IP}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-32B", "messages": [{"role": "user", "content": "Hello"}]}'

# Request to LoRA adapter (routes to Qwen pool)
curl -X POST "http://${GATEWAY_IP}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{"model": "food-review-1", "messages": [{"role": "user", "content": "Review this restaurant"}]}'

# Request to DeepSeek base model
curl -X POST "http://${GATEWAY_IP}/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{"model": "deepseek/DeepSeek-r1", "messages": [{"role": "user", "content": "Solve this problem"}]}'
```

## Troubleshooting

### Check IPP Logs

```bash
kubectl logs -n ${NAMESPACE} -l app=payload-processor --tail=100
```

### Verify ConfigMap Discovery

IPP should log discovered model mappings at startup. Check that your ConfigMaps have the required label:

```bash
kubectl get configmap -l inference.llm-d.ai/ipp-managed=true -n ${NAMESPACE}
```

### Verify Routing

Check EPP logs to confirm requests reach the correct pool:

```bash
# Check Qwen pool
kubectl logs -n ${NAMESPACE} -l inferencepool=qwen-pool-epp --tail=20

# Check DeepSeek pool
kubectl logs -n ${NAMESPACE} -l inferencepool=deepseek-pool-epp --tail=20
```

## Cleanup

```bash
# Remove IPP
helm uninstall ipp -n ${NAMESPACE}

# Remove InferencePools
helm uninstall qwen-pool -n ${NAMESPACE}
helm uninstall deepseek-pool -n ${NAMESPACE}

# Remove routes and ConfigMaps
kubectl delete httproute qwen-route deepseek-route -n ${NAMESPACE}
kubectl delete configmap -l inference.llm-d.ai/ipp-managed=true -n ${NAMESPACE}

# Remove namespace
kubectl delete namespace ${NAMESPACE}

# Clean up cloned repo
rm -rf /tmp/ipp
```

## Further Reading

* [Multi-Model Routing Capability](../../docs/well-lit-paths/foundations/multi-model-routing.md) — High-level overview and architecture
* [IPP Architecture](../../docs/architecture/advanced/inference-payload-processing/README.md) — Technical details of the Inference Payload Processor
* [IPP Repository](https://github.com/llm-d/llm-d-inference-payload-processor) — Source code, configuration reference, and plugin documentation
