# vLLM Demo

 - Tested on Kubernetes Version: v1.33.13
 - Single node cluster
 - 4 x NVIDIA L4
 - vLLM: `registry.redhat.io/rhaiis/vllm-cuda-rhel9:3.2.0`
 - Model: `ibm-granite/granite-4.2-8b`

This repo is intended to be a tutorial on how to run vLLM in Kubernetes powered by NVIDA accelerators. 
It includes a number of iterative steps that build on eachother.

1. Accelerator lookup - How to look GPU type in your cluster
2. Basic deployment of vLLM in k8s - Minimal deployment
3. Adding a model cache layer for faster (re)deployments - Storing a model from Hugging Face in a persistant storage layer
4. Adding ingress - Adding ingress via Gateway API with routing rules to different models (for future MaaS functionalit)
5. More being developed (see `TODO` at the bottom) - Performance Tuning, OGX, RAG, llm-d, Governance, etc...

## vLLM on Kubernetes via Red Hat AI Inference Server (RHAIIS)

 Red Hat AI Inference provides enterprise-grade stability and security for serving large language 
 models across hybrid cloud and edge environments. Built on the open source vLLM project, AI 
 Inference delivers optimized standalone inference and Kubernetes-native distributed inference to 
 meet the demands of production AI workloads. 

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

TODO: 
- Performance tuning
- OGX (formerly Llama Stack)
- RAG integration
- llm-d
- multi model
- Auth tokens
- Token quotas