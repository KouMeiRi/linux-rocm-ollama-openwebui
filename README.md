# linux-docker-rocm-ollama-openwebui
## Local LLM & Web UI Stack (Ollama + Open-WebUI) with AMD ROCm on Linux (Fedora, Ubuntu, Debian, etc.).

*Updated: August 2026*

This repository provides an optimized Docker Compose configuration for running a fully local AI agent stack (Ollama LLM engine + Open WebUI frontend) on **Linux distributions (Fedora, Ubuntu, Debian, etc.)**, utilizing **AMD GPU (ROCm) hardware acceleration**.

### Key Problems Solved
- **ROCm Container Passthrough**: Configures Kernel Fusion Driver (`/dev/kfd`) and Render Direct Infrastructure (`/dev/dri`) for containerized AMD GPU support.
- **RDNA GPU Architecture Spoofing**: Bypasses driver incompatibility issue on unsupported AMD GPUs via `HSA_OVERRIDE_GFX_VERSION`.
- **Isolated Network Architecture**: Connects backend LLM services and Web UI within a dedicated bridge network without exposure conflicts.

---

## 🛠️ Docker Compose Configuration

`docker-compose.yml`

```yaml
services:
  ollama:
    image: ollama/ollama:rocm
    container_name: ollama
    restart: unless-stopped
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    environment:
      - HSA_OVERRIDE_GFX_VERSION=10.3.0
      - OLLAMA_FLASH_ATTENTION=1
      - OLLAMA_KV_CACHE_TYPE=q8_0
      - GPU_MAX_ALLOC_PERCENT=100
      - GPU_SINGLE_ALLOC_PERCENT=100
      - OLLAMA_KEEP_ALIVE=5m
      - OLLAMA_NUM_PARALLEL=2
      - TZ=Asia/Tokyo
    volumes:
      - /path/to/your/ollama_data:/root/.ollama
    ports:
      - "11434:11434"
    networks:
      - custom-ai-net

  openwebui:
    image: ghcr.io/open-webui/open-webui:latest
    container_name: openwebui
    restart: unless-stopped
    environment:
      - TZ=Asia/Tokyo
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - /path/to/your/openwebui_config:/app/config
      - /path/to/your/openwebui_data:/app/backend/data
    ports:
      - "3000:8080"
    networks:
      - custom-ai-net
    depends_on:
      - ollama

networks:
  custom-ai-net:
    driver: bridge
    name: custom-ai-net
```


📋 Configuration Notes & Customization
1. ⚠️ GPU Architecture Setting (HSA_OVERRIDE_GFX_VERSION) — CRITICAL

The value 10.3.0 is configured for AMD RDNA2 GPUs (e.g., RX 6800 / RX 6700 XT). Do not skip this step! If you are using a different GPU series (such as RDNA3 / RX 7000 series), you MUST modify this variable to match your target GFX architecture version (e.g., 11.0.0), otherwise ROCm will fail to initialize the GPU inside the container.

設定値 10.3.0 は AMD RDNA2 GPU（例：RX 6800 / RX 6700 XT）向けの設定です。動作に直結するため必ず確認してください。 RDNA3（RX 7000 シリーズなど）や異なるアーキテクチャの GPU を使用する場合は、自身のグラフィックボードに対応する GFX バージョン（例：11.0.0）に変更する必要があります。変更しない場合、コンテナ内で GPU 加速が正常に認識されません。

2. Volume Paths (volumes)

Replace /path/to/your/... with your actual host directories.
Ensure the Docker user has write permissions.

📌 Important Update (2024+ OpenWebUI versions)
OpenWebUI no longer uses /data for backend storage.
All persistent data (chat history, embeddings, uploads, vector DB, Whisper models) is stored under:
```
/app/backend/data
```
Therefore, the correct volume mapping is:
```
- /path/to/your/openwebui_data:/app/backend/data
```
旧バージョンで使用されていた /data は現在利用されません。
チャット履歴・RAG・アップロードファイルなどの永続データは すべて /app/backend/data に保存されます。

3. Network Configuration (custom-ai-net)

custom-ai-net is a custom bridge network defined for internal container communication. You can replace custom-ai-net with any network name suitable for your infrastructure.

custom-ai-net はコンテナ間の内部通信用に定義されたカスタムブリッジネットワークです。環境の命名規則に合わせて任意のネットワーク名に変更可能です。

4. Timezone (TZ)

Set to Asia/Tokyo by default. Adjust to your local timezone string if necessary.

デフォルトで Asia/Tokyo に設定されています。必要に応じてお住まいの地域のタイムゾーン（例：UTC, America/New_York）に変更してください。

<img width="1591" height="431" alt="demo" src="https://github.com/user-attachments/assets/f59547ec-a771-47e8-aad7-a91a2c47baf5" />

5. 🚀 Ollama Runtime Optimizations (2026 features)

These environment variables enable advanced inference optimizations in Ollama:

| Variable | Description |
|---|---|
| `OLLAMA_FLASH_ATTENTION=1` | Enables flash attention for faster inference |
| `OLLAMA_KV_CACHE_TYPE=q8_0` | Uses 8-bit quantized KV cache to reduce VRAM usage |
| `GPU_MAX_ALLOC_PERCENT=100` | Allows ROCm to allocate up to 100% of GPU memory |
| `GPU_SINGLE_ALLOC_PERCENT=100` | Allows single allocation to use full GPU memory |
| `OLLAMA_KEEP_ALIVE=5m` | Keeps models loaded in VRAM for 5 minutes |
| `OLLAMA_NUM_PARALLEL=2` | Allows up to 2 parallel inference requests |

⚠️ **Warning**: `OLLAMA_NUM_PARALLEL=2` may reduce single-query speed. 
If you prioritize fast individual responses over throughput, set this to `1`.

⚠️ 注意: `OLLAMA_NUM_PARALLEL=2` は単一クエリの速度を低下させる可能性があります。
スループットより個別応答速度を重視する場合は `1` に設定してください。

---

## 📥 Model Management & Initial Setup

To download and run LLM models inside the Ollama container, execute the following command:

```bash
docker exec -it ollama ollama pull gemma4:12b
```
English: Replace gemma4:12b with any model tag available on Ollama Library.

日本語: 「gemma4:12b」 の部分を、利用したいモデル名（Ollama Library 上のタグ）に変更して実行してください。

<img width="2252" height="1288" alt="page" src="https://github.com/user-attachments/assets/e05dfd93-ed98-40bc-b403-4d688e62b1e5" />


⚡ Performance Optimization (Open WebUI Settings)
To reduce unnecessary LLM API calls and optimize response speeds on local GPUs, it is highly recommended to disable background auto-generation features.

Recommended Settings
Navigate to Settings -> Interface (設定 -> インターフェース) and turn OFF the following options:

Title Auto Generation / タイトル自動生成

Follow-up Questions Auto Generation / 関連質問の自動生成

Chat Tags Auto Generation / チャットタグの自動生成

💡 Why Disable These?

By default, Open WebUI triggers background LLM tasks after every message to generate chat titles, tags, and suggested follow-ups. Disabling these options prevents background queue congestion, noticeably improving single-query generation speeds on local AMD ROCm setups.

Open WebUI の初期設定では、メッセージ送信ごとにバックグラウンドでタイトル・タグ・関連質問生成の推論タスクが自動運行されます。これらをオフにすることで余計な GPU リソース消費を抑え，ローカル環境（AMD ROCm）での推論応答速度を大幅に向上させることができます。

<img width="1607" height="1075" alt="settings" src="https://github.com/user-attachments/assets/fc81c6e6-136a-4834-bc79-f3c5b0fb58e2" />




