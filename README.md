# vLLM Demo

 - Tested on Kubernetes Version: v1.33.13
 - Single node cluster
 - 4 and 1 x NVIDIA L4
   - (`g6.8xlarge` = single L4)
   - (`g6.12xlarge` = four L4)
 - vLLM: `registry.redhat.io/rhaii/vllm-cuda-rhel9:3.5.1`
 - Model: `ibm-granite/granite-4.2-8b`

This repo is intended to be a tutorial on how to run vLLM in Kubernetes powered by NVIDA accelerators. 
It includes a number of iterative steps that build on eachother.

1. Accelerator lookup - How to look GPU type in your cluster
2. Basic deployment of vLLM in k8s - Minimal deployment
3. Adding a model cache layer for faster (re)deployments - Storing a model from Hugging Face in a persistant storage layer
4. Adding ingress - Adding ingress via Gateway API with routing rules to different models (for future MaaS functionality)
5. More being developed (see `TODO` at the bottom) - Performance Tuning, OGX, RAG, llm-d, Governance, etc...

## vLLM on Kubernetes via Red Hat AI Inference Server (RHAIIS)

 This demo is using Red Hat AI Inference Server (RHAIIS). RHAIIS provides enterprise-grade stability and security for serving 
 large language models across hybrid cloud and edge environments. Built on the open source vLLM project, AI 
 Inference delivers optimized standalone inference and Kubernetes-native distributed inference to meet the demands of 
 production AI workloads. 

## Prerequisites

- You have installed the kubectl cli.
- You have logged in as a user with cluster-admin privileges.
- You have installed NFD and the required GPU Operator for your underlying AI accelerator hardware.


## Accelerator lookup

```bash
kubectl get nodes -l nvidia.com/gpu.present=true -o custom-columns=NAME:.metadata.name,PRODUCT:.metadata.labels."nvidia\.com/gpu\.product"
NAME                                             PRODUCT
ip-###-#-##-##.ap-northeast-1.compute.internal   NVIDIA-L4
```

## Granite Model Registry on Huggingface

IBM Granite models can be found at:   
`https://huggingface.co/ibm-granite`

IBM's Granite models on Hugging Face are released under Apache 2.0 and are not gated, so you don't 
need a token/login just to download them. You can pull them anonymously.

Set a `HF_TOKEN` to enable higher rate limits and faster downloads.

## Simple deployment of vLLM on Kubernetes

1. Create a namespace for the deployment

```bash
kubectl create namespace rhaiis-demo 
```

2. Deploy vLLM to namespace (deployment and service)

```bash
kubectl apply -f k8s/vllm/deployment.yaml -n rhaiis-demo   
```

Output:
```
deployment.apps/rhaiis-granite created
service/rhaiis-granite created
```

Quick sanity check to confirm it actually serves a request:

```bash
kubectl port-forward svc/rhaiis-granite 8000:8000 -n rhaiis-demo
```

then in another terminal:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "ibm-granite/granite-4.2-8b", "messages": [{"role": "user", "content": "Hello"}]}'
```

Sample output:
```json
{
  "id": "chatcmpl-245777c0037f4bfbb7f39470d5910a7c",
  "object": "chat.completion",
  "created": 1789167499,
  "model": "ibm-granite/granite-4.2-8b",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "reasoning_content": null,
        "content": "Okay, the user said \"Hello\". I should respond politely. Keep it friendly and open-ended to encourage them to ask their question.\n</think>\nHello! 😊 How can I help you today? Feel free to ask me any question—whether it's about science, tech, writing, everyday advice, or just something curious on your mind. What would you like to explore?",
        "tool_calls": []
      },
      "logprobs": null,
      "finish_reason": "stop",
      "stop_reason": null
    }
  ],
  "usage": {
    "prompt_tokens": 16,
    "total_tokens": 95,
    "completion_tokens": 79,
    "prompt_tokens_details": null
  },
  "prompt_logprobs": null,
  "kv_transfer_params": null
}
```

## Persistant storage layer for your models

You may have noticed that it took some time for vllm to get up and running. 
Not only did k8 pull the container image for vLLM, but also the model from hugging face, and these can be quite large.

Every time we redeploy, or roll out a new pod we would have to wait for these each time, so lets add a PVC to store
the model, so subsequent redeployments take less time.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rhaiis-cache
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3-csi
  resources:
    requests:
      storage: 50Gi
```

in in our deployment

```yaml
...
          volumeMounts:
            - name: cache
              mountPath: /opt/app-root/src/.cache
...
      volumes:
        - name: cache
          persistentVolumeClaim:
            claimName: rhaiis-cache
...
```
to apply these changes

```bash
kubectl apply -f k8s/vllm/deployment-w-storage.yaml -n rhaiis-demo 
```
Output
```bash
persistentvolumeclaim/rhaiis-cache created
deployment.apps/rhaiis-granite configured
service/rhaiis-granite unchanged
```

3-5+ minutes the first time deploying, but if you do a rollout/restart:

```bash
kubectl rollout restart deployment -n rhaiis-demo 
```
 you will see the redeployment time drop significatly (90 seconds in my test environment)

## Adding Ingress via Gateway API

Gateway API is becoming the new standard for ingress on k8s. You may be using something else, and
that is fine.

Gateway API is also a great way to provide Model-as-a-Service (MaaaS), since you can run multiple
inferences servers in your cluster, and use routing rules (such as path or header evaluation) to 
send traffic to different models (in differnt namespaces if needed) with a common entry point.

1. Create a namespace for the gateway

```bash
kubectl create namespace maas-gateway 
```

2. TLS for the Gateway

The `maas-gateway` Gateway terminates HTTPS, which means it needs a TLS certificate before its
listener will accept any traffic. We use are using cert-manager's **self-signed Issuer** to generate 
one:

```bash
kubectl apply -f k8s/ingress/certs.yaml -n maas-gateway  
```

output
```
issuer.cert-manager.io/maas-gateway-selfsigned created
certificate.cert-manager.io/maas-gateway-tls created
```

3. Create the Gateway

```bash
kubectl apply -f k8s/ingress/gateway.yaml -n maas-gateway
```

Once applied, get the Gateway's external address with:
```bash
kubectl get gateway maas-gateway -n maas-gateway -o jsonpath='{.status.addresses[0].value}'
```

4. Create an HTTPRoute with header based routing rules, in case we want to add more vllm servers and 
provide access through the gateway

```bash
kubectl apply -f k8s/vllm/httproute.yaml -n rhaiis-demo  
```

Test access to the model

```bash
GATEWAY=$(kubectl get gateway maas-gateway -n maas-gateway -o jsonpath='{.status.addresses[0].value}')\

echo https://$GATEWAY/v1/chat/completions

curl -k https://$GATEWAY/v1/chat/completions \
  -H "x-model-name: granite-4.2-8b" \
  -H "Content-Type: application/json" \
  -d '{"model": "ibm-granite/granite-4.2-8b", "messages": [{"role": "user", "content": "Hello"}]}' | jq
```

Try will a different/invalide value of `x-model-name` to get a 404 error

Note: TLS and DNS can be automated with other tools, such as the upstream project `kuadrant` or 
`Red Hat Connectivity Link`

## Performance Tuning

Run a benchmark on our running model

```bash
kubectl run vllm-bench --rm -i --restart=Never -n rhaiis-demo \
  --image=registry.redhat.io/rhaii/vllm-cuda-rhel9:3.5.1 \
  --env="HF_HUB_OFFLINE=0" \
  --command -- vllm bench serve \
    --backend openai-chat \
    --base-url http://rhaiis-granite:8000 \
    --endpoint /v1/chat/completions \
    --model ibm-granite/granite-4.2-8b \
    --dataset-name random --random-input-len 512 --random-output-len 128 --num-prompts 50 --max-concurrency 1 
```
Sample Output

```bash
...
Starting initial single prompt test run...
Skipping endpoint ready check.
Starting main benchmark run...
Traffic request rate: inf
Burstiness factor: 1.0 (Poisson process)
Maximum request concurrency: 1
100%|██████████| 50/50 [07:15<00:00,  8.71s/it]
tip: install termplotlib and gnuplot to plot the metrics
============ Serving Benchmark Result ============
Successful requests:                     50        
Failed requests:                         0         
Maximum request concurrency:             1         
Benchmark duration (s):                  435.42    
Total input tokens:                      26350     
Total generated tokens:                  6400      
Request throughput (req/s):              0.11      
Output token throughput (tok/s):         14.70     
Peak output token throughput (tok/s):    16.00     
Peak concurrent requests:                2.00      
Total token throughput (tok/s):          75.22     
---------------Time to First Token----------------
Mean TTFT (ms):                          237.68    
Median TTFT (ms):                        240.25    
P99 TTFT (ms):                           246.88    
-----Time per Output Token (excl. 1st token)------
Mean TPOT (ms):                          66.70     
Median TPOT (ms):                        66.71     
P99 TPOT (ms):                           66.77     
---------------Inter-token Latency----------------
Mean ITL (ms):                           66.17     
Median ITL (ms):                         66.69     
P99 ITL (ms):                            69.24     
==================================================
```

Key Metics :
- `TTFT` (Time to First Token) — how long a user waits before anything starts appearing. This is what makes an app feel "responsive" vs. "stuck." Dominated by prefill time plus queueing time if the server is saturated.
- `TPOT` / `ITL` (Time per Output Token / Inter-Token Latency) — how fast tokens stream in after the first one. This is what makes output feel "fast" vs. "typing slowly." Roughly, 1000/TPOT = tokens/sec per stream.
- `Throughput` (req/s, tok/s) — total system capacity, aggregated across all concurrent requests.
- `Peak concurrent requests` — how many requests the server had in flight at once.


### Max conncurency

Increase the value `--max-concurrency 1` (+) to see how that affects metrics

In my environment as I increase concurrency from 1 to 20, I noticed two things:

1. Individual latency degrades linearly with load — expected. GPU compute for prefill is a shared, serialized resource; more concurrent requests means more queueing ahead of each new one.

2. Aggregate throughput keeps improving — total time to finish 50 requests dropped from 7m15s (concurrency 1) to 39s (concurrency 20). The GPU is doing more useful work per second at higher concurrency, even though each individual request waits longer. This is the textbook LLM-serving latency/throughput tradeoff, and the data shows it working exactly as expected — no cliff, no thrashing, no pathological blowup. I haven't even found the ceiling yet; the line is still climbing straight at concurrency=20.

Conncurrency 30-60

- Total time for 50 requests goes flat at ~29-30s starting at `concurrency 30`, and stays there through 60. That's the `throughput ceiling` — the server literally cannot push more total work through per second no matter how much client-side concurrency you throw at it beyond this point. This is the definitive signal predicted last time.

with `max concurrency` of 30

```bash
Maximum request concurrency: 30
100%|██████████| 50/50 [00:29<00:00,  1.68it/s]
tip: install termplotlib and gnuplot to plot the metrics
============ Serving Benchmark Result ============
Successful requests:                     50        
Failed requests:                         0         
Maximum request concurrency:             30        
Benchmark duration (s):                  29.82     
Total input tokens:                      26350     
Total generated tokens:                  6400      
Request throughput (req/s):              1.68      
Output token throughput (tok/s):         214.62    
Peak output token throughput (tok/s):    360.00    
Peak concurrent requests:                42.00     
Total token throughput (tok/s):          1098.27   
---------------Time to First Token----------------
Mean TTFT (ms):                          2459.21   
Median TTFT (ms):                        2149.76   
P99 TTFT (ms):                           5154.68   
-----Time per Output Token (excl. 1st token)------
Mean TPOT (ms):                          102.91    
Median TPOT (ms):                        108.38    
P99 TPOT (ms):                           118.70    
---------------Inter-token Latency----------------
Mean ITL (ms):                           102.10    
Median ITL (ms):                         86.61     
P99 ITL (ms):                            661.68    
==================================================
```

Tuning the models server itself

flags to set in your deployment

`--kv-cache-dtype fp8` — directly attacks the wall you just found

Your benchmark data showed the ceiling is KV-cache-capacity-bound (~30 concurrent sequences, 19,744 tokens available ÷ ~640 tokens/request). Quantizing the KV cache to FP8 roughly halves the bytes per token stored, which — for the exact same GPU memory budget — roughly doubles that ceiling to ~60 before you'd hit the same wall again. This doesn't touch model weights or accuracy of the weights themselves, only how KV cache entries are stored; there's a small numerical-precision tradeoff on attention, but it's normally negligible for serving.

```yaml
- "--kv-cache-dtype"
- "fp8"
```

Worth re-running the same concurrency sweep after adding this — you should see the same linear region persist further out before the knee shows up (predict it around ~60 instead of ~30).

`--max-num-batched-tokens` — smooths the compute-bound region (1-20)

This controls the chunked-prefill budget per scheduler step. Raising it from the (likely still-default) 2048 lets more prefill work land in a single step instead of queueing sequentially across steps — this is what's producing that clean ~85ms-per-request slope you measured in the linear region. Worth experimenting with:

```yaml
- "--max-num-batched-tokens"
- "4096"
```

Tradeoff: larger prefill chunks can add latency to decode steps for requests already streaming (prefill and decode compete for the same batch step budget), so this one benefits from being benchmarked, not just assumed.

`--swap-space` — optional safety valve, not a real fix

Default is 4GiB of CPU RAM available for KV cache to spill into when GPU KV cache is full, instead of immediately queueing. This can smooth the transition zone (your 30-40 knee) by letting a few extra sequences continue rather than hard-queueing, at the cost of PCIe transfer latency for the swapped sequences. It doesn't raise the real ceiling, just changes how ungracefully you fall off it — the FP8 KV cache change is the one that actually moves the wall.

```bash
kubectl apply -f k8s/vllm/deployment-performance.yaml -n rhaiis-demo 
```
New Benchmarks show:
- Ceiling: `~60-70` concurrent requests (up from `~30-40` pre-fp8-KV)
- Saturated throughput: ~2.0 req/s / ~410 tok/s sustained
- Saturated latency: Mean `TTFT` ~3.6-4.0s, `P99` ~4.9-5.3s, stable under overload rather than blowing up
- Cost: none on weights, minor KV-cache quantization overhead at low concurrency (the slight mean/median regression you saw at exactly concurrency=30 earlier)

## Model weight quantization (FP8)

By default the model loads in bf16 — the full-precision format IBM publishes. Switching to the `-fp8` variant (`ibm-granite/granite-4.2-8b-fp8`) quantizes the model weights to 8-bit floating point, which the L4's Ada Lovelace architecture supports natively in hardware.

This is a different lever than the `--kv-cache-dtype fp8` flag above: KV cache quantization affects how many concurrent requests fit in memory, while weight quantization affects how fast each request's prefill/decode computation actually runs. The two stack together.

Tradeoff: unlike KV cache quantization (generally lossless for output quality), weight quantization is a real reduction in numerical precision. For most text generation this is negligible, but worth spot-checking outputs after switching — especially for reasoning/tool-calling use cases, which can be more sensitive to precision loss than plain generation.

To deploy with the new model:

```bash
kubectl apply -f k8s/vllm/deployment-f8-optimized.yaml -n rhaiis-demo 
```

## Performance tuning results

Benchmarked with `vllm bench serve` (random dataset, 512 input / 128 output tokens, 50 requests) across a range of `--max-concurrency` values, on a single NVIDIA L4.

### Mean TTFT by configuration

| Concurrency | bf16 (baseline) | bf16 + `--kv-cache-dtype fp8` | **fp8 weights + `--kv-cache-dtype fp8`** |
|---|---|---|---|
| 1 | 238ms | — | **147ms** |
| 20 | 1857ms | — | **170ms** |
| 30 | 2463ms | 2576ms | **192ms** |
| 40 | 4439ms | 3063ms | **256ms** |
| 60 | 7820ms | 3658ms | **320ms** |
| Saturated throughput | ~1.66 req/s (hard cliff, P99 ~20s) | ~2.0 req/s (graceful plateau) | **~6.5 req/s** |

### What each tweak fixed

- **`--kv-cache-dtype fp8`** raised the *capacity ceiling* — more concurrent requests fit in the same GPU memory, and the server degrades gracefully under overload instead of hitting a catastrophic latency cliff (P99 TTFT no longer spikes to 15-20s past saturation).

- **FP8 model weights** (`ibm-granite/granite-4.2-8b-fp8`) fixed the *compute cost itself* — Ada Lovelace's native FP8 tensor cores cut prefill time per request dramatically. The TTFT growth rate per added concurrent request dropped from ~85ms/request (bf16) to ~1ms/request — a ~70x flatter curve — while also freeing enough memory to push the throughput ceiling to ~6.5 req/s, over 3x the KV-cache-only result.

Together, these took mean TTFT at 30 concurrent requests from **2463ms → 192ms**, roughly a 13x improvement, with sustained throughput more than tripling.

**Caveat:** FP8 weight quantization is a real reduction in numerical precision, unlike KV cache quantization which is effectively lossless for output quality. Worth spot-checking outputs (especially tool-calling / reasoning prompts) before treating this as production-ready — not yet formally validated here.

## OGX (formerly Llama stack)

### Why OGX?

Your vLLM deployment already serves an OpenAI-compatible API — OGX sits in front of it as an orchestration layer, not a replacement. Three concrete reasons to add it:

- **Model-agnostic client compatibility.** OGX exposes model aliases that decouple what a client asks for from what's actually serving it — in this demo, requests for `claude-haiku-4-5-20251001` transparently route to Granite underneath. Client code doesn't need to know or care which model/provider is actually behind the API.

- **Tool-calling and agent orchestration, without hand-building the loop.** Granite 4.2 supports tool-calling natively, but using that capability directly means implementing the multi-step tool-call loop yourself. OGX moves that orchestration server-side.

- **A path to RAG.** Built-in vector store and file-search providers (`faiss`, `sqlite-vec`) mean adding retrieval-augmented generation is a config change, not a new subsystem to build.

It's an extra moving part for a "just serve a model" demo, but the right layer once you want to build an actual application — agent or RAG — on top of the inference server rather than just benchmark it.

```bash
# deploy OGX
kubectl apply -f k8s/ogx/deployment-base.yaml -n rhaiis-demo

# add it to our gateway
kubectl apply -f k8s/ogx/http-route.yaml -n rhaiis-demo 

# test access to it
GATEWAY=$(kubectl get gateway maas-gateway -n maas-gateway -o jsonpath='{.status.addresses[0].value}')\

echo https://$GATEWAY/v1/chat/completions

curl -k https://$GATEWAY/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "vllm/ibm-granite/granite-4.2-8b-fp8", "messages": [{"role": "user", "content": "Hello"}]}' | jq
```

Note: we removed the routing header, and are now only relying on the specified model in the payload data. 
we also added the `vllm/` prefix to the model specification, which is how OGX is routing to our running model.

For this demo we have one model server (`vllm`) and one `OGX` instance running in the same namespace, but in production, like the ingress gateway, OGX would run in it's own namespace and route traffic to different runing models, based on payload data `model` specification.

Example architexture:

```
                    ┌─────────────────────────────────┐
                    │  Shared Gateway (Istio-backed)   │
                    │  GatewayClass: openshift-ai-     │
                    │  inference/istio                 │
                    └────────────────┬──────────────────┘
                                     │
                    ┌────────────────▼──────────────────┐
                    │  ai-gateway-operator layer:        │
                    │  rate limiting, API keys,          │
                    │  subscription tiers, auth          │
                    │  (this is the piece replacing      │
                    │   our "no auth for now")           │
                    └────────────────┬──────────────────┘
                                     │
                    ┌────────────────▼──────────────────┐
                    │  OGX (own namespace) — model-      │
                    │  agnostic routing/orchestration,   │
                    │  same role it plays in this demo,  │
                    │  just N backends instead of 1      │
                    └───┬──────────┬──────────┬──────────┘
                        │          │          │
                 ┌──────▼───┐ ┌───▼──────┐ ┌─▼────────┐
                 │ granite- │ │ llama-3  │ │ (model N)│
                 │ ns       │ │ ns       │ │ ns       │
                 │ (RHAIIS) │ │ (RHAIIS) │ │ ...      │
                 └──────────┘ └──────────┘ └──────────┘
```

TODO: 

- OGX (formerly Llama Stack)
- RAG integration
- llm-d
- multi model
- Auth tokens
- Token quotas