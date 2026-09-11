# vLLM on Kubernetes via Red Hat AI Inference Server (RHAIIS)

 Red Hat AI Inference provides enterprise-grade stability and security for serving large language models across hybrid cloud and edge environments. Built on the open source vLLM project, AI Inference delivers optimized standalone inference and Kubernetes-native distributed inference to meet the demands of production AI workloads. 

 - Tested on Kubernetes Version: v1.33.13
 - Single node cluster
 - 4 x NVIDIA L4
 - RHAIIS: `registry.redhat.io/rhaiis/vllm-cuda-rhel9:3.2.0`
 - Model: `ibm-granite/granite-4.2-8b`

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

IBM's Granite models on Hugging Face are released under Apache 2.0 and are not gated, so you don't need a token/login just to download them. You can pull them anonymously.

## Simple deployment of vLLM on Kubernetes

1. Create a namespace for the deployment

```bash
kubectl create namespace raiis-demo 
```

2. Deploy vLLM to namespace (deployment and service)

```bash
kubectl apply -f k8s/vllm/deployment.yaml -n raiis-demo   
```

Output:
```
deployment.apps/rhaiis-granite created
service/rhaiis-granite created
```

Quick sanity check to confirm it actually serves a request:

```bash
kubectl port-forward svc/rhaiis-granite 8000:8000 -n raiis-demo
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