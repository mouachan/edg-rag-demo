# EDG RAG Demo - Model Deployment

This repository contains the Kubernetes manifests for deploying AI models on OpenShift AI with KServe.

## Models

### 1. Mistral 3 14B Instruct
A generative model for chat and RAG use cases.
- Model: `mistralai/Ministral-3-14B-Instruct-2512`
- Storage: 50Gi
- GPUs: 2x NVIDIA L40

### 2. Gemma 3 Embedding (300M)
An embedding model for text vectorization.
- Model: `google/embeddinggemma-300m`
- Storage: 10Gi
- GPUs: 1x NVIDIA L40

## Prerequisites

- OpenShift cluster with OpenShift AI installed
- Access to the `edg-demo` namespace
- NVIDIA GPU nodes with L40 GPUs
- HuggingFace account with access token

## Deployment Steps

### 1. Create HuggingFace Secret

First, create the HuggingFace token secret:

```bash
# Edit the secret file and replace YOUR_HUGGINGFACE_TOKEN_HERE with your actual token
vi models/huggingface-secret.yaml

# Apply the secret
oc apply -f models/huggingface-secret.yaml
```

### 2. Deploy Mistral 3 14B Instruct

```bash
cd models/mistral-3-14b-instruct

# 1. Create the PVC for model storage
oc apply -f pvc.yaml

# 2. Download the model using a Job
oc apply -f download-job.yaml

# Wait for the job to complete (this may take 30-60 minutes)
oc get jobs -n edg-demo -w

# 3. Create the ServingRuntime
oc apply -f servingruntime.yaml

# 4. Deploy the InferenceService
oc apply -f inferenceservice.yaml

# 5. Wait for the InferenceService to be ready
oc get inferenceservice mistral-3-14b-instruct -n edg-demo -w
```

### 3. Deploy Gemma 3 Embedding

```bash
cd models/gemma-3-embedding

# 1. Create the PVC for model storage
oc apply -f pvc.yaml

# 2. Download the model using a Job
oc apply -f download-job.yaml

# Wait for the job to complete (this may take 5-10 minutes)
oc get jobs -n edg-demo -w

# 3. Create the ServingRuntime
oc apply -f servingruntime.yaml

# 4. Deploy the InferenceService
oc apply -f inferenceservice.yaml

# 5. Wait for the InferenceService to be ready
oc get inferenceservice embeddinggemma-300m -n edg-demo -w
```

## Verification

### Check Model Status

```bash
# Check all InferenceServices
oc get inferenceservices -n edg-demo

# Check running pods
oc get pods -n edg-demo

# Check routes
oc get routes -n edg-demo
```

### Test Mistral 3 14B Instruct

```bash
MISTRAL_URL=$(oc get route mistral-3-14b-instruct -n edg-demo -o jsonpath='{.spec.host}')

curl -X POST https://$MISTRAL_URL/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mistral-3-14b-instruct",
    "messages": [{"role": "user", "content": "Hello, how are you?"}],
    "temperature": 0.7
  }'
```

### Test Gemma 3 Embedding

```bash
GEMMA_URL=$(oc get route embeddinggemma-300m -n edg-demo -o jsonpath='{.spec.host}')

curl -X POST https://$GEMMA_URL/embed \
  -H "Content-Type: application/json" \
  -d '{
    "inputs": "This is a test sentence for embedding"
  }'
```

## Cleanup

To remove all resources:

```bash
# Delete Mistral resources
oc delete -f models/mistral-3-14b-instruct/inferenceservice.yaml
oc delete -f models/mistral-3-14b-instruct/servingruntime.yaml
oc delete -f models/mistral-3-14b-instruct/download-job.yaml
oc delete -f models/mistral-3-14b-instruct/pvc.yaml

# Delete Gemma resources
oc delete -f models/gemma-3-embedding/inferenceservice.yaml
oc delete -f models/gemma-3-embedding/servingruntime.yaml
oc delete -f models/gemma-3-embedding/download-job.yaml
oc delete -f models/gemma-3-embedding/pvc.yaml

# Delete the HuggingFace secret
oc delete -f models/huggingface-secret.yaml
```

## Troubleshooting

### Model Download Issues

If the download job fails:

```bash
# Check job logs
oc logs job/mistral-model-download -n edg-demo
# or
oc logs job/gemma-embedding-download -n edg-demo

# Common issues:
# - Invalid HuggingFace token
# - Network connectivity issues
# - Insufficient storage
```

### InferenceService Not Ready

```bash
# Check predictor pod logs
oc logs -l app=isvc.mistral-3-14b-instruct-predictor -n edg-demo
# or
oc logs -l app=isvc.embeddinggemma-300m-predictor -n edg-demo

# Check events
oc get events -n edg-demo --sort-by='.lastTimestamp'

# Common issues:
# - GPU not available
# - Model files not found in PVC
# - OOM (Out of Memory) errors
```

## Resource Requirements

### Mistral 3 14B Instruct
- **CPU**: 8 cores
- **Memory**: 16Gi
- **GPU**: 2x NVIDIA L40
- **Storage**: 50Gi (PVC)
- **Zone**: eu-central-1a

### Gemma 3 Embedding
- **CPU**: 4 cores
- **Memory**: 16Gi
- **GPU**: 1x NVIDIA L40
- **Storage**: 10Gi (PVC)
- **Zone**: eu-central-1a

## Architecture

```
┌─────────────────────────────────────────────────┐
│        OpenShift / OpenShift AI Platform        │
├─────────────────────────────────────────────────┤
│                                                 │
│  ┌───────────────────┐   ┌──────────────────┐  │
│  │  Mistral 3 14B    │   │  Gemma Embedding │  │
│  │  Instruct         │   │  300M            │  │
│  │                   │   │                  │  │
│  │  ┌─────────────┐  │   │  ┌────────────┐  │  │
│  │  │ vLLM Runtime│  │   │  │ TEI Runtime│  │  │
│  │  └─────────────┘  │   │  └────────────┘  │  │
│  │                   │   │                  │  │
│  │  ┌─────────────┐  │   │  ┌────────────┐  │  │
│  │  │ PVC (50Gi) │  │   │  │ PVC (10Gi) │  │  │
│  │  └─────────────┘  │   │  └────────────┘  │  │
│  └───────────────────┘   └──────────────────┘  │
│                                                 │
│  ┌───────────────────────────────────────────┐  │
│  │     HuggingFace Token Secret              │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

## License

This project is part of the EDG RAG Demo.
