---
title: Envoy AI Gateway
weight: 1
---

[Envoy AI Gateway](https://aigateway.envoyproxy.io/) is an open source project for using Envoy Gateway
to handle request traffic from application clients to Generative AI services.

## How to use

### Enable Envoy Gateway and Envoy AI Gateway

Both of them are already enabled by default in `values.global.yaml` and will be deployed in llmaz-system.

```yaml
envoy-gateway:
    enabled: true
envoy-ai-gateway:
    enabled: true
```

However, [Envoy Gateway](https://gateway.envoyproxy.io/latest/install/install-helm/) and [Envoy AI Gateway](https://aigateway.envoyproxy.io/docs/getting-started/) can be deployed standalone in case you want to deploy them in other namespaces.

### Basic Example

To expose your models via Envoy Gateway, you need to create a GatewayClass, Gateway, and AIGatewayRoute. The following example shows how to do this.

We'll deploy two models `Qwen/Qwen2-0.5B-Instruct-GGUF` and `Qwen/Qwen2.5-Coder-0.5B-Instruct-GGUF` with llama.cpp (cpu only) and expose them via Envoy AI Gateway.

The full example is [here](https://github.com/InftyAI/llmaz/blob/main/docs/examples/envoy-ai-gateway/basic.yaml), apply it.

```bash
kubectl apply -f https://raw.githubusercontent.com/InftyAI/llmaz/refs/heads/main/docs/examples/envoy-ai-gateway/basic.yaml
```

### Query AI Gateway APIs

If Open-WebUI is enabled, you can chat via the webui (recommended), see [documentation](./open-webui.md). Otherwise, following the steps below to test the Envoy AI Gateway APIs.

I. Port-forwarding the `LoadBalancer` service in llmaz-system, like:

```bash
kubectl -n llmaz-system port-forward \
  $(kubectl -n llmaz-system get svc \
    -l gateway.envoyproxy.io/owning-gateway-name=default-envoy-ai-gateway \
    -o name) \
  8080:80
```

II. Query `curl http://localhost:8080/v1/models | jq .`, available models will be listed. Expected response will look like this:

```json
{
  "data": [
    {
      "id": "qwen2-0.5b",
      "created": 1745327294,
      "object": "model",
      "owned_by": "Envoy AI Gateway"
    },
    {
      "id": "qwen2.5-coder",
      "created": 1745327294,
      "object": "model",
      "owned_by": "Envoy AI Gateway"
    }
  ],
  "object": "list"
}
```

III. Query `http://localhost:8080/v1/chat/completions` to chat with the model. Here, we ask the `qwen2-0.5b` model, the query will look like:

```bash
curl -H "Content-Type: application/json"     -d '{
        "model": "qwen2-0.5b",
        "messages": [
            {
                "role": "system",
                "content": "Hi."
            }
        ]
    }'     http://localhost:8080/v1/chat/completions | jq .
```

Expected response will look like this:

```json
{
  "choices": [
    {
      "finish_reason": "stop",
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello! How can I assist you today?"
      }
    }
  ],
  "created": 1745327371,
  "model": "qwen2-0.5b",
  "system_fingerprint": "b5124-bc091a4d",
  "object": "chat.completion",
  "usage": {
    "completion_tokens": 10,
    "prompt_tokens": 10,
    "total_tokens": 20
  },
  "id": "chatcmpl-AODlT8xnf4OjJwpQH31XD4yehHLnurr0",
  "timings": {
    "prompt_n": 1,
    "prompt_ms": 319.876,
    "prompt_per_token_ms": 319.876,
    "prompt_per_second": 3.1262114069201816,
    "predicted_n": 10,
    "predicted_ms": 1309.393,
    "predicted_per_token_ms": 130.9393,
    "predicted_per_second": 7.63712651587415
  }
}
```
