# RAG on Kubernetes with Qwen3-8B-Instruct (vLLM) + Qdrant

Single-GPU (RTX 4090) RAG stack:

```
                 +----------------------+
                 |      Ingress         |  (optional)
                 +----------+-----------+
                            |
                    rag-api-service
                            |
                     RAG API (FastAPI)
                     - embeds queries (sentence-transformers, CPU)
                     - retrieves context from Qdrant
                     - calls vLLM for generation
          +-----------------+-----------------+
          |                                   |
      vllm-service                     qdrant-service
   Qwen3-8B-Instruct (vLLM)               Qdrant
          |                                   |
     RTX 4090 GPU node                Persistent Volume
```

## Prerequisites

1. A Kubernetes cluster with a node that has the RTX 4090 attached, and the
   NVIDIA device plugin / GPU Operator installed so the node exposes the
   `nvidia.com/gpu` resource.
2. Label the GPU node:
   ```bash
   kubectl label node <node-name> gpu=rtx4090
   ```
3. `kubectl` configured against your cluster, and Docker for building the
   RAG API image.

## Deploy

```bash
kubectl apply -f kubernetes/00-namespace.yaml
# Optional, only if Qwen3-8B-Instruct becomes gated/rate-limited for you:
# kubectl create secret generic hf-secret --namespace rag-system --from-literal=HF_TOKEN=<token>
kubectl apply -f kubernetes/02-storage.yaml
kubectl apply -f kubernetes/10-vllm-deployment.yaml
kubectl apply -f kubernetes/11-vllm-service.yaml
kubectl apply -f kubernetes/20-qdrant-deployment.yaml
kubectl apply -f kubernetes/21-qdrant-service.yaml
```

Build and push the RAG API image, then update the `image:` field in
`kubernetes/30-rag-api-deployment.yaml`:

```bash
cd rag-api
docker build -t myrepo/rag-api:v1 .
docker push myrepo/rag-api:v1
cd ..
```

```bash
kubectl apply -f kubernetes/30-rag-api-deployment.yaml
kubectl apply -f kubernetes/31-rag-api-service.yaml
# Optional, only if you have an ingress controller:
# kubectl apply -f kubernetes/40-ingress.yaml
```

## Watch the vLLM pod come up

The first startup downloads Qwen3-8B-Instruct (~16 GB) from Hugging Face into
the `hf-cache-pvc`, so it takes a few minutes the first time. Subsequent
restarts reuse the cache.

```bash
kubectl -n rag-system get pods -w
kubectl -n rag-system logs deployment/vllm -f
```

## Ingest documents

Port-forward Qdrant and run the ingestion script locally (or `kubectl cp` it
into a pod):

```bash
kubectl -n rag-system port-forward svc/qdrant-service 6333:6333 &
pip install qdrant-client sentence-transformers
python rag-api/ingest.py --docs-dir ./my-docs --recreate
```

## Query the RAG API

```bash
kubectl -n rag-system port-forward svc/rag-api-service 8080:80 &

curl -X POST http://localhost:8080/query \
  -H "Content-Type: application/json" \
  -d '{"question": "What does the onboarding doc say about VPN access?"}'
```

## Notes / tuning for a single RTX 4090 (24 GB)

- `--gpu-memory-utilization 0.90` and `--max-model-len 16384` in
  `10-vllm-deployment.yaml` are safe starting points for an 8B model in
  bfloat16 on 24 GB. Lower `max-model-len` further if you see OOM errors, or
  raise it if you have headroom and need longer contexts.
- The RAG API's embedding model (`all-MiniLM-L6-v2`) runs on CPU inside the
  `rag-api` pods — it's small and doesn't need the GPU, which is reserved
  for vLLM.
- `rag-api` is scaled to 2 replicas since it's cheap (CPU-only); `vllm` stays
  at 1 replica since only one pod can use the single GPU.
- Set `storageClassName` in `02-storage.yaml` to match a storage class that
  exists on your cluster (or remove the line to use the default).
