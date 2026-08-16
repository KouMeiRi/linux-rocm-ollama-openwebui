[English](./README.md) | [日本語](./README.ja.md) | [繁體中文](./README.zh-tw.md)

---

# linux-docker-rocm-ollama-openwebui
## Linux (Fedora, Ubuntu, Debian など) 上の AMD ROCm を使用したローカル LLM & Web UI スタック (Ollama + Open-WebUI)

*更新日: 2026年8月*

本リポジトリは、**Linux ディストリビューション (Fedora, Ubuntu, Debian など)** 上で、**AMD GPU (ROCm) ハードウェアアクセラレーション**を活用し、完全ローカルの AI エージェントスタック (Ollama LLM エンジン + Open WebUI フロントエンド) を実行するための最適化された Docker Compose 構成を提供します。

### 解決する主な課題
- **ROCm コンテナパススルー**: コンテナ化された AMD GPU サポートのために、Kernel Fusion Driver (`/dev/kfd`) と Render Direct Infrastructure (`/dev/dri`) を構成します。
- **RDNA GPU アーキテクチャのスプーフィング**: `HSA_OVERRIDE_GFX_VERSION` を介して、サポートされていない AMD GPU でのドライバー非互換性問題を回避します。
- **分離されたネットワークアーキテクチャ**: 公開の競合なく、専用のブリッジネットワーク内でバックエンド LLM サービスと Web UI を接続します。

---

## 🛠️ Docker Compose 構成

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


📋 構成に関する注意事項とカスタマイズ
1. ⚠️ GPU アーキテクチャ設定 (HSA_OVERRIDE_GFX_VERSION) — 最重要

設定値 10.3.0 は AMD RDNA2 GPU（例：RX 6800 / RX 6700 XT）向けの設定です。動作に直結するため必ず確認してください。 RDNA3（RX 7000 シリーズなど）や異なるアーキテクチャの GPU を使用する場合は、自身のグラフィックボードに対応する GFX バージョン（例：11.0.0）に変更する必要があります。変更しない場合、コンテナ内で GPU 加速が正常に認識されません。

2. ボリュームパス (volumes)

`/path/to/your/...` を実際のホストディレクトリに置き換えてください。Docker ユーザーに書き込み権限があることを確認してください。

📌 重要なアップデート (2024年以降の OpenWebUI バージョン)
OpenWebUI はバックエンドストレージに `/data` を使用しなくなりました。すべての永続データ（チャット履歴、埋め込み、アップロード、ベクター DB、Whisper モデル）は以下の場所に保存されます：
```
/app/backend/data
```
したがって、正しいボリュームマッピングは次のとおりです：
```
- /path/to/your/openwebui_data:/app/backend/data
```

3. ネットワーク構成 (custom-ai-net)

custom-ai-net はコンテナ間の内部通信用に定義されたカスタムブリッジネットワークです。環境の命名規則に合わせて任意のネットワーク名に変更可能です。

4. タイムゾーン (TZ)

デフォルトで Asia/Tokyo に設定されています。必要に応じてお住まいの地域のタイムゾーン（例：UTC, America/New_York）に変更してください。

<img width="1591" height="431" alt="demo" src="https://github.com/user-attachments/assets/f59547ec-a771-47e8-aad7-a91a2c47baf5" />

5. 🚀 Ollama ランタイム最適化 (2026年の新機能)

これらの環境変数は、Ollama の高度な推論最適化を有効にします：

| 変数 | 説明 |
|---|---|
| `OLLAMA_FLASH_ATTENTION=1` | Flash Attention を有効化し、推論を高速化します |
| `OLLAMA_KV_CACHE_TYPE=q8_0` | 8ビット量子化 KV キャッシュを使用し、VRAM 使用量を削減します |
| `GPU_MAX_ALLOC_PERCENT=100` | ROCm に GPU メモリの最大 100% を割り当てることを許可します |
| `GPU_SINGLE_ALLOC_PERCENT=100` | 単一の割り当てでフル GPU メモリを使用することを許可します |
| `OLLAMA_KEEP_ALIVE=5m` | モデルを VRAM に 5 分間ロードしたまま保持します |
| `OLLAMA_NUM_PARALLEL=2` | 最大 2 つの並列推論リクエストを許可します |

⚠️ **注意**: `OLLAMA_NUM_PARALLEL=2` は単一クエリの速度を低下させる可能性があります。スループットより個別応答速度を重視する場合は `1` に設定してください。

---

## 📥 モデル管理と初期設定

Ollama コンテナ内で LLM モデルをダウンロードして実行するには、次のコマンドを実行します：

```bash
docker exec -it ollama ollama pull gemma4:12b
```
「gemma4:12b」 の部分を、利用したいモデル名（Ollama Library 上のタグ）に変更して実行してください。

<img width="2252" height="1288" alt="page" src="https://github.com/user-attachments/assets/e05dfd93-ed98-40bc-b403-4d688e62b1e5" />


⚡ パフォーマンス最適化 (Open WebUI 設定)
不要な LLM API 呼び出しを減らし、ローカル GPU での応答速度を最適化するために、バックグラウンドの自動生成機能を無効にすることを強くお勧めします。

推奨設定
Settings -> Interface (設定 -> インターフェース) に移動し、以下のオプションを OFF にします：

- Title Auto Generation / タイトル自動生成
- Follow-up Questions Auto Generation / 関連質問の自動生成
- Chat Tags Auto Generation / チャットタグの自動生成

💡 なぜこれらを無効にするのか？

Open WebUI の初期設定では、メッセージ送信ごとにバックグラウンドでタイトル・タグ・関連質問生成の推論タスクが自動運行されます。これらをオフにすることで余計な GPU リソース消費を抑え，ローカル環境（AMD ROCm）での推論応答速度を大幅に向上させることができます。

<img width="1607" height="1075" alt="settings" src="https://github.com/user-attachments/assets/fc81c6e6-136a-4834-bc79-f3c5b0fb58e2" />
