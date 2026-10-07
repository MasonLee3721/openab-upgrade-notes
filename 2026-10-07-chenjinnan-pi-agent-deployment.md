# OpenAB 新增第 9 位 Agent「陳近南」部署與排錯實錄 (Pi Coding Agent + ChatGPT OAuth)

> **日期**：2026-10-07  
> **環境**：k3s (`vm-3-21-ubuntu`), Helm v3, OpenAB 0.9.0  
> **角色**：陳近南（天地會總舵主 / 架構宗師）  
> **目標**：在 `agent-broker` 命名空間新增第 9 位 Agent，完成鹿鼎記全員陣容（韋小寶、蘇荃、方怡、阿珂、建寧、沐劍屏、曾柔、雙兒、陳近南）之 Discord 聯動，並成功串接 ChatGPT 訂閱帳號之 OAuth 認證。

---

## 一、 技術棧與架構設計

1. **容器映像檔**：
   - 基底：`ghcr.io/openabdev/openab:0.9.0-codex`
   - 客製安裝：`@earendil-works/pi-coding-agent@1.0.4`、`@ccgv2/pi-acp@0.6.0`
   - 本地標籤：`openab-pi:latest`（利用 k3s containerd socket 直接建置並載入命名空間 `k8s.io`）
2. **通訊協定**：
   - OpenAB ↔ Agent：Agent Client Protocol (ACP) via stdio
   - OpenAB ↔ Discord：Discord Gateway (Bot Token / Mentions 觸發)
   - OpenAB ↔ STT：Groq Whisper (`whisper-large-v3-turbo`)
3. **模型與認證**：
   - 認證方式：OpenAI ChatGPT OAuth（Codex endpoint，支援訂閱帳戶使用）
   - 執行模型：`gpt-5.5`（medium thinking）
4. **持久化與狀態**：
   - PVC：`agent-broker-openab-chenjinnan` (1Gi, local-path)
   - 存儲路徑：`/home/node`（包含 `.pi/agent/auth.json`、`.openab/`、對話 session 等）

---

## 二、 核心踩坑歷程與技術根因 (Pitfalls & Troubleshooting)

### 踩坑 1：節點資源緊繃導致 Pod Pending (`Insufficient memory`)
* **現象**：
  新建立之陳近南 Pod 卡在 `Pending` 狀態無法啟動。
  ```bash
  kubectl describe pod ...
  # 0/1 nodes are available: 1 Insufficient memory. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.
  ```
* **排查**：
  叢集為單節點 k3s 主機，既有 8 位 Agent 各設定 `requests.memory: 256Mi`，節點的可分配記憶體已達臨界值，無額外 256Mi 提供調度。
* **解法**：
  調整 Helm values，將 `chenjinnan` 的記憶體請求調低至 `requests.memory: 128Mi`（上限 `limits: 1500Mi` 維持不變），成功通過調度器檢查並正常 Running。

---

### 踩坑 2：ChatGPT OAuth Device Flow 不支援與遠端回呼攔截
* **現象**：
  嘗試使用傳統 CLI Device Code 流程：
  ```bash
  kubectl exec -it ... -- pi /login --use-device-flow
  # 提示：Device flow only supports Anthropic provider.
  ```
  OpenAI 的 OAuth 認證強制啟動本機 HTTP 伺服器並監聽 `http://127.0.0.1:1455/auth/callback`。然而 Pod 運行在遠端 Kubernetes 內，瀏覽器在外部點擊授權後無法直接連線至 Pod 內部回呼網址。
* **解法**：
  1. 在本地瀏覽器打開授權 URL 登入 ChatGPT，點擊授權。
  2. 瀏覽器重定向至 `http://127.0.0.1:1455/auth/callback?code=...&state=...` 時會顯示連線失敗（此為預期情況）。
  3. 複製瀏覽器網址列中完整的重定向 URL。
  4. 在 Pod 內使用 `curl` 模擬回呼請求：
     ```bash
     kubectl exec -it deployment/agent-broker-openab-chenjinnan -- \
       curl -s "http://127.0.0.1:1455/auth/callback?code=...&state=..."
     ```
  5. 驗證成功，OAuth 憑證自動寫入 `/home/node/.pi/agent/auth.json`。

---

### 踩坑 3：ChatGPT 訂閱帳號的模型限制 (Model Constraint)
* **現象**：
  在 Pod 內測試 `pi --model gpt-4o -p "測試"` 時報錯：
  ```text
  The 'gpt-4o' model is not supported when using Codex with a ChatGPT account.
  ```
* **根因**：
  OpenAI 官方對 ChatGPT OAuth 帳號（Codex endpoint）限制僅支援特定的 Codex / 推理模型，一般 `gpt-4o` 會被 API 端點拒絕。
* **解法**：
  改用 `gpt-5.5` 推理模型，並在 `/home/node/.pi/agent/settings.json` 設定 `"defaultModel": "gpt-5.5"`。測試 CLI `pi -p "你是誰"` 順利獲得陳近南的宗師風格回應。

---

### 踩坑 4（核心大坑）：Discord 出現空回覆警示，`pi-acp` 遺失 API Key
* **現象**：
  在 Discord `@陳近南` 發送訊息時，Bot 固定回傳：
  ```text
  ⚠️ The agent did not produce a response. This usually indicates a backend configuration issue — not an intentional empty reply. Please try again later.
  ```
  OpenAB Pod 日誌記錄：
  ```text
  WARN openab_core::adapter: agent returned empty turn (0 output tokens) — likely provider/model/auth failure stop_reason=Some("end_turn")
  ```
* **排查**：
  1. **確認非 Discord 設定問題**：此警示訊息為 OpenAB 收到 0 token 時透過 Discord API 發出的，代表 Bot Token、Gateway 連線、收發權限皆完全正常。
  2. **調閱 Session 日誌**：檢查 `/home/node/.pi/agent/sessions/...jsonl`，抓到真實錯誤訊息：
     ```json
     {"role":"assistant","content":[],"stopReason":"error","errorMessage":"No API key for provider: openai"}
     ```
  3. **追查依賴版本差異**：
     - CLI `pi` 命令使用的是全域的 `@earendil-works/pi-coding-agent@1.0.4`，內建讀取 `auth.json` 自動注入 OAuth Bearer Token。
     - OpenAB 啟動的 `@ccgv2/pi-acp` (v0.6.0) 在其內部巢狀 node_modules 中自帶了舊版 `@earendil-works/pi-coding-agent@0.75.3`。該舊版本在調用 OpenAI API 時**強制檢查環境變數 `OPENAI_API_KEY`**，無法自動掛載 OAuth Token。
* **解法**：
  採用與專案中既有 Agent（如小寶使用之 `agy-acp-wrapper.sh`）相同的架構模式，建立專屬包裝腳本 `/home/node/pi-acp-wrapper.sh`：
  ```bash
  #!/bin/bash
  # 自動透過 CLI 取得當前有效的 OAuth Token（過期自動觸發 Refresh）
  TOKEN=$(pi auth print-bearer-token --provider openai 2>/dev/null)
  if [ -n "$TOKEN" ]; then
      export OPENAI_API_KEY="$TOKEN"
  fi
  exec /usr/local/bin/pi-acp "$@"
  ```
  賦予執行權限後，在 Helm values 中將 Agent 啟動指令設定為：
  ```yaml
  chenjinnan:
    command: /home/node/pi-acp-wrapper.sh
    args: []
  ```
  在 Pod 內模擬 ACP 協定發送提示，確認 `pi-acp` 順利產出 Token 串流與完整回應。

---

### 踩坑 5：歷史對話與 Session 失敗狀態快取
* **現象**：
  即使後端修復完成，針對同一個 Discord Thread 發話，仍偶爾出現重現失敗的問題。
* **根因**：
  OpenAB 會將 Discord Thread ID 快取於 `/home/node/.openab/cache/threads.json`，並透過 `session/load` 重新載入先前失敗的舊 Session ID。
* **解法**：
  在部署重啟時清除該快取：
  ```bash
  rm -f /home/node/.openab/cache/threads.json
  ```
  強制 OpenAB 在收到下一次提及時，建立全新的 ACP Session。

---

### 踩坑 6：人設關係與稱謂細緻化
* **調整**：
  陳近南初版提示詞由其他夫人模板繼承而來，原記載「家主 / 老公」。為符合《鹿鼎記》天地會總舵主、一代宗師的人設氣度，將對最高決策者（MasonLee / `<@1331833906751869030>`）之稱謂修正為**「先生 / Mason兄」**，以知己與摯友之禮相待，言談沉穩肅穆，不失長者風範。

---

## 三、 驗證與現行狀態

1. **鹿鼎記 9 人全員正常就緒**：
   - 韋小寶 (`xiaobao`)
   - 蘇荃 (`suquan`)
   - 方怡 (`fangyi`)
   - 阿珂 (`ake`)
   - 建寧公主 (`jianning`)
   - 沐劍屏 (`jianping`)
   - 曾柔 (`zengrou`)
   - 雙兒 (`shuaner`)
   - 陳近南 (`chenjinnan`)
2. **相互提及相容**：
   全數 9 位 Agent 的 `trustedBotIds` 均已交叉配置完成，支援彼此互相 `@` 協同與轉發任務。
3. **Helm 版本演進**：
   最終配置保存於 `/home/ubuntu/values-helm-chenjinnan-final.yaml`，目前運行於 Revision 53，各 Pod 均為 `1/1 Running`。
