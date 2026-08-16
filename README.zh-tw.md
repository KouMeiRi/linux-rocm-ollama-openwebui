[English](./README.md) | [日本語](./README.ja.md) | [繁體中文](./README.zh-tw.md)

---

# linux-docker-rocm-ollama-openwebui
## 在 Linux (Fedora, Ubuntu, Debian 等) 上使用 AMD ROCm 的本地 LLM 與 Web UI 堆疊 (Ollama + Open-WebUI)

*更新日期：2026 年 8 月*

本Repo提供了一套最佳化的 Docker Compose 設定，用於在 **Linux 發行版 (Fedora, Ubuntu, Debian 等)** 上，利用 **AMD GPU (ROCm) 硬體加速**，運行完全本地的 AI 代理堆疊 (Ollama LLM 引擎 + Open WebUI 前端)。

### 解決的核心問題
- **ROCm 容器直通 (Passthrough)**：設定 Kernel Fusion Driver (`/dev/kfd`) 與 Render Direct Infrastructure (`/dev/dri`)，以支援容器內的 AMD GPU。
- **RDNA GPU 架構偽裝**：透過 `HSA_OVERRIDE_GFX_VERSION` 繞過不支援的 AMD GPU 驅動程式不相容問題。
- **隔離的網路架構**：在專用的橋接網路中連接後端 LLM 服務與 Web UI，避免連接埠衝突。

---

## 🛠️ Docker Compose 設定

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


📋 設定注意事項與自訂
1. ⚠️ GPU 架構設定 (HSA_OVERRIDE_GFX_VERSION) — 極度重要

數值 10.3.0 是為 AMD RDNA2 GPU（例如：RX 6800 / RX 6700 XT）所設定。請勿跳過此步驟！如果您使用的是其他 GPU 系列（例如 RDNA3 / RX 7000 系列），您**必須**修改此變數以符合目標 GFX 架構版本（例如：11.0.0），否則 ROCm 將無法在容器內初始化 GPU。

2. 儲存區路徑 (volumes)

將 `/path/to/your/...` 替換為您實際的主機目錄。請確保 Docker 使用者具有寫入權限。

📌 重要更新 (2024+ OpenWebUI 版本)
OpenWebUI 不再使用 `/data` 作為後端儲存。所有持久性資料（對話歷史、向量嵌入、上傳檔案、向量資料庫、Whisper 模型）皆儲存於：
```
/app/backend/data
```
因此，正確的儲存區映射為：
```
- /path/to/your/openwebui_data:/app/backend/data
```

3. 網路設定 (custom-ai-net)

`custom-ai-net` 是為容器內部通訊所定義的自訂橋接網路。您可以將 `custom-ai-net` 替換為適合您基礎設施的任何網路名稱。

4. 時區 (TZ)

預設設定為 Asia/Tokyo。如有必要，請調整為您當地的時區字串（例如：UTC, America/New_York）。

<img width="1591" height="431" alt="demo" src="https://github.com/user-attachments/assets/f59547ec-a771-47e8-aad7-a91a2c47baf5" />

5. 🚀 Ollama 執行環境最佳化 (2026 年新特性)

這些環境變數可啟用 Ollama 中的進階推論最佳化：

| 變數 | 說明 |
|---|---|
| `OLLAMA_FLASH_ATTENTION=1` | 啟用 Flash Attention 以加快推論速度 |
| `OLLAMA_KV_CACHE_TYPE=q8_0` | 使用 8-bit 量化 KV 快取以減少 VRAM 使用量 |
| `GPU_MAX_ALLOC_PERCENT=100` | 允許 ROCm 配置高達 100% 的 GPU 記憶體 |
| `GPU_SINGLE_ALLOC_PERCENT=100` | 允許單次配置使用完整的 GPU 記憶體 |
| `OLLAMA_KEEP_ALIVE=5m` | 將模型在 VRAM 中保留 5 分鐘 |
| `OLLAMA_NUM_PARALLEL=2` | 允許最多 2 個並行推論請求 |

⚠️ **警告**：`OLLAMA_NUM_PARALLEL=2` 可能會降低單一查詢的速度。如果您更看重單次回應的速度而非整體吞吐量，請將其設定為 `1`。

---

## 📥 模型管理與初始設定

若要在 Ollama 容器內下載並執行 LLM 模型，請執行以下指令：

```bash
docker exec -it ollama ollama pull gemma4:12b
```
將 `gemma4:12b` 替換為 Ollama Library 上任何可用的模型標籤。

<img width="2252" height="1288" alt="page" src="https://github.com/user-attachments/assets/e05dfd93-ed98-40bc-b403-4d688e62b1e5" />


⚡ 效能最佳化 (Open WebUI 設定)
為了減少不必要的 LLM API 呼叫並最佳化本地 GPU 的回應速度，強烈建議停用背景自動生成功能。

建議設定
導覽至 Settings -> Interface (設定 -> 介面)，並關閉以下選項：

- Title Auto Generation / 標題自動生成
- Follow-up Questions Auto Generation / 後續問題自動生成
- Chat Tags Auto Generation / 對話標籤自動生成

💡 為什麼要停用這些功能？

預設情況下，Open WebUI 會在每次傳送訊息後觸發背景 LLM 任務，以生成對話標題、標籤和建議的後續問題。停用這些選項可防止背景佇列塞車，顯著提升本地 AMD ROCm 環境下的單一查詢生成速度。

<img width="1607" height="1075" alt="settings" src="https://github.com/user-attachments/assets/fc81c6e6-136a-4834-bc79-f3c5b0fb58e2" />
