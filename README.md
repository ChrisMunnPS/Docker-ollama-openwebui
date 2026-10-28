# Ollama + Open WebUI (NVIDIA GPU)

Local AI stack with **Ollama** (LLM backend) and **Open WebUI** (ChatGPT-like interface), configured for NVIDIA GPUs.

---

## Requirements

- Docker + Docker Compose
- NVIDIA driver
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)

Verify GPU access:

```bash
nvidia-smi
```

---

## Quick Start

1. Create a project folder and save the `compose.yaml` below into it.

2. Create the data directories:

```bash
mkdir -p ollama/data openwebui/data
```

3. Start the stack:

```bash
docker compose up -d
```

4. Open the UI:

- Open WebUI → http://localhost:3000
- Ollama API → http://localhost:11434

---

## compose.yaml

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    environment:
      - OLLAMA_CONTEXT_LENGTH=8192
      - OLLAMA_KEEP_ALIVE=10m
      - OLLAMA_FLASH_ATTENTION=1
      - OLLAMA_KV_CACHE_TYPE=q4_0
    volumes:
      - ./ollama/data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]

  openwebui:
    image: ghcr.io/open-webui/open-webui:cuda
    container_name: openwebui
    restart: unless-stopped
    ports:
      - "3000:8080"
    volumes:
      - ./openwebui/data:/app/backend/data
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - USE_CUDA_DOCKER=true
    depends_on:
      - ollama
```

> **Note:** The `:cuda` image allows Open WebUI’s own embedding / reranking models to use the GPU (much faster RAG/search).

---

## Common Commands

All commands below should be run from the folder that contains `compose.yaml`.

### Start / Stop / Restart

```bash
# Start
docker compose up -d

# Stop
docker compose down

# Restart everything
docker compose restart

# Restart only one service
docker compose restart ollama
docker compose restart openwebui
```

### View Logs

```bash
# Both services
docker compose logs -f

# Only Ollama
docker compose logs -f ollama

# Only Open WebUI
docker compose logs -f openwebui
```

### Check Status

```bash
docker compose ps
```

### Update Images

```bash
docker compose pull
docker compose up -d
```

---

## Managing Models

### List installed models

```bash
docker compose exec ollama ollama list
```

### Pull (download) a model

```bash
docker compose exec ollama ollama pull qwen2.5-coder:14b
docker compose exec ollama ollama pull qwen3:14b
docker compose exec ollama ollama pull nomic-embed-text
docker compose exec ollama ollama pull qwen3:8b
```

### Remove a model

```bash
docker compose exec ollama ollama rm MODEL_NAME

# Examples
docker compose exec ollama ollama rm llama3.2
docker compose exec ollama ollama rm deepseek-r1:14b
```

### Show model info

```bash
docker compose exec ollama ollama show qwen2.5-coder:14b
```

---

## Recommended Models (12 GB VRAM)

| Model                  | Size   | Use Case                              |
|------------------------|--------|---------------------------------------|
| `qwen2.5-coder:14b`    | ~9 GB  | Best coding / PowerShell / HA YAML    |
| `qwen3:14b`            | ~9.3 GB| Strong general + tool use             |
| `qwen3:8b`             | ~5.2 GB| Fast secondary model                  |
| `nomic-embed-text`     | ~0.3 GB| RAG / document search (required)      |

---

## Useful One-Liners

```bash
# Enter Ollama container shell
docker compose exec ollama sh

# Check GPU is visible inside Ollama
docker compose exec ollama nvidia-smi

# Force recreate containers (after changing compose.yaml)
docker compose up -d --force-recreate
```

---

## Tips

- After changing the `compose.yaml`, always run:
  ```bash
  docker compose up -d
  ```
- Models are stored in `./ollama/data`
- Open WebUI data (chats, settings, knowledge bases) is stored in `./openwebui/data`
- For better RAG performance, set the embedding model in Open WebUI to **nomic-embed-text** (Admin → Settings → Documents)

---

## Troubleshooting

**"no configuration file provided"**  
→ Make sure you are in the folder that contains `compose.yaml`.

**"exec: ollama: executable file not found"**  
→ The `ollama` service is probably using the wrong image. Confirm the image is `ollama/ollama:latest`.

**Slow document search**  
→ Use the `:cuda` image for Open WebUI and set embedding engine to Ollama + `nomic-embed-text`.
