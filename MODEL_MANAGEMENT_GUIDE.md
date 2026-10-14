# OpenClaw Model Management Guide (Ollama & Local Models)

This guide walks you through the exact steps to add, switch, and set default models in OpenClaw when using local models with Ollama.

---

## 1. Architecture Overview

- **Host (Mac)**: Runs Ollama (`ollama serve` / `ollama run ...`) listening on `127.0.0.1:11434`.
- **OpenClaw Gateway**: Runs inside a Docker container (`q-claw-openclaw-gateway-1`).
- **Connection URL**: Inside Docker, `127.0.0.1` refers to the container itself. To reach Ollama on the Mac host, OpenClaw connects via:
  ```
  http://host.docker.internal:11434
  ```
- **API Key**: Local Ollama does not require real credentials. OpenClaw uses `ollama-local` as a synthetic marker in `.env` and configuration.

---

## 2. Step-by-Step: Adding and Defaulting a New Ollama Model

### Step 1: Run or pull your model in Ollama on the host
```bash
# Pull or run any model from Ollama or HuggingFace
ollama run hf.co/unsloth/Qwen3.8-27B-GGUF:UD-Q3_K_XL
```

Verify the exact model name from your terminal:
```bash
ollama list
```

---

### Step 2: Check model discovery and connection from OpenClaw
Run this from the `Q-Claw` workspace:
```bash
docker exec q-claw-openclaw-gateway-1 curl -s http://host.docker.internal:11434/api/tags
```
You should see your model listed in the JSON output.

---

### Step 3: Add the model to OpenClaw and set it as default

You can apply configuration updates live without restarting the Gateway using `openclaw config patch`:

```bash
docker exec -i q-claw-openclaw-gateway-1 node dist/index.js config patch --stdin << 'EOF'
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://host.docker.internal:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: [
          {
            id: "YOUR_OLLAMA_MODEL_ID",
            name: "YOUR_DISPLAY_NAME",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 65536,
            contextTokens: 65536,
            maxTokens: 8192,
            params: {
              num_ctx: 65536,
              keep_alive: "15m"
            }
          }
        ]
      }
    }
  },
  agents: {
    defaults: {
      model: {
        primary: "ollama/YOUR_OLLAMA_MODEL_ID",
        fallbacks: [
          "digitalocean/qwen3.5-397b-a17b",
          "digitalocean/glm-5.3-flash"
        ]
      },
      models: {
        "ollama/YOUR_OLLAMA_MODEL_ID": {
          alias: "your-short-alias"
        }
      }
    }
  }
}
EOF
```

*Note: Replace `YOUR_OLLAMA_MODEL_ID`, `YOUR_DISPLAY_NAME`, and `your-short-alias` with your model's details.*

---

### Step 4: Manage Aliases & Defaults via CLI

OpenClaw also provides direct CLI commands to manage aliases and switch defaults:

#### 1. Add or change an alias:
```bash
docker exec q-claw-openclaw-gateway-1 node dist/index.js models aliases add <alias-name> ollama/<model-id>
```
*Example:*
```bash
docker exec q-claw-openclaw-gateway-1 node dist/index.js models aliases add qwen-local-3xl ollama/hf.co/unsloth/Qwen3.8-27B-GGUF:UD-Q3_K_XL
```

#### 2. Change the default primary model:
```bash
docker exec q-claw-openclaw-gateway-1 node dist/index.js models set ollama/<model-id>
```

#### 3. View configured models and aliases:
```bash
docker exec q-claw-openclaw-gateway-1 node dist/index.js models status
docker exec q-claw-openclaw-gateway-1 node dist/index.js models list
docker exec q-claw-openclaw-gateway-1 node dist/index.js models aliases list
```

---

## 3. Switching Models in Active Chat Sessions

When chatting with OpenClaw in any connected channel (Web Control UI, Discord, Telegram, etc.):

- **Switch session to a specific model or alias:**
  ```text
  /model qwen-local-3xl
  ```
  or
  ```text
  /model ollama/hf.co/unsloth/Qwen3.8-27B-GGUF:UD-Q3_K_XL
  ```

- **Reset session to the configured default model:**
  ```text
  /model default
  ```

- **List available models in chat:**
  ```text
  /models
  ```

---

## 4. Verifying End-to-End Inference

You can run a quick one-shot test inference through OpenClaw at any time to confirm that the model responds properly:

```bash
docker exec q-claw-openclaw-gateway-1 node dist/index.js infer model run --prompt "Reply with exactly: test ok"
```

Expected output:
```text
model.run via local
provider: ollama
model: <your-configured-model>
outputs: 1
test ok
```

---

## 5. Context Window Sizing & "Conversation too long" Fix

If a session returns `"This conversation is too long for the model. Try /compact, or start a new conversation with /new."`:

1. **In-Chat Fixes:**
   - Type `/compact` to summarize older messages while keeping recent turns intact.
   - Type `/new` to start a fresh conversation context.

2. **Hardware & Sizing Guidance (Apple Silicon Unified Memory):**
   - **32,768 (32k tokens)**: Lightweight, but fills up fast on long agent tasks.
   - **65,536 (64k tokens - Currently Active)**: Balanced sweet spot for 27B models on 24GB unified memory. Prompt evaluation takes < 1 second.
   - **131,072 (128k tokens)**: Fits maximum history, but KV cache demands may cause memory swapping and slower generation on 24GB RAM.

To adjust the context window, change `contextWindow`, `contextTokens`, and `params.num_ctx` in the config patch from Step 3.

---

## 6. Summary Cheat Sheet

| Action | Command |
|---|---|
| Check Ollama models on host | `ollama list` |
| View OpenClaw model status | `docker exec q-claw-openclaw-gateway-1 node dist/index.js models status` |
| Set a new default model | `docker exec q-claw-openclaw-gateway-1 node dist/index.js models set <provider/model>` |
| Add a model alias | `docker exec q-claw-openclaw-gateway-1 node dist/index.js models aliases add <alias> <provider/model>` |
| Test model inference | `docker exec q-claw-openclaw-gateway-1 node dist/index.js infer model run --prompt "test"` |
| Switch model in chat | `/model <alias>` or `/model default` |
