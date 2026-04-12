# OpenAB Helm 升級紀錄

> 日期：2026-04-12
> 升級版本：`agent-broker-0.3.3` → `openab-0.7.0`

---

## 環境

- Kubernetes：k3s
- Helm：v3.20.1
- Release 名稱：`agent-broker`
- Namespace：`agent-broker`

---

## 升級步驟

### 1. 加入新 Helm repo

舊 repo（`agent-broker`）已 404，改用新的：

```bash
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm repo add openab https://openabdev.github.io/openab
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm repo update
```

### 2. 查看目前 values

```bash
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm get values agent-broker -n agent-broker
```

> 注意：舊版用 `discord.botToken`，新版改為 `agents.kiro.discord.botToken`

### 3. 執行升級

```bash
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm upgrade agent-broker openab/openab \
  -n agent-broker \
  --set agents.kiro.discord.botToken="<BOT_TOKEN>" \
  --set-string 'agents.kiro.discord.allowedChannels[0]=<CHANNEL_ID>'
```

### 4. 確認版本

```bash
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm list -n agent-broker
```

---

## 踩坑：升級後 Discord bot 出現 `connection lost`

### 原因

新版 Chart 改了 values 結構（多包一層 `agents.kiro`），PVC 名稱從 `openab` 變成 `agent-broker-openab-kiro`。Helm 把舊 PVC 刪掉，kiro-cli 的 OAuth token 跟著消失。

### 解法：重新登入 kiro-cli

```bash
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml \
  kubectl exec -it deployment/agent-broker-openab-kiro -n agent-broker \
  -- kiro-cli login --use-device-flow
```

登入完成後 restart pod：

```bash
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml \
  kubectl rollout restart deployment/agent-broker-openab-kiro -n agent-broker
```

---

## 防禦：避免下次升級再遇到 PVC 資料消失

### 加上 keep annotation（已執行）

```bash
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml \
  kubectl annotate pvc agent-broker-openab-kiro -n agent-broker \
  helm.sh/resource-policy=keep
```

這樣 Helm 升級時不會刪除這個 PVC。

### 升級前確認 PVC 名稱沒變

```bash
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml \
  helm template agent-broker openab/openab \
  --set agents.kiro.discord.botToken="<BOT_TOKEN>" \
  --set-string 'agents.kiro.discord.allowedChannels[0]=<CHANNEL_ID>' \
  | grep -i "kind: PersistentVolumeClaim" -A5
```

確認名稱仍是 `agent-broker-openab-kiro` 再升級。

---

## 常用指令

```bash
# 查版本
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml helm list -n agent-broker

# 查 pod 狀態
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubectl get pods -n agent-broker

# 查 log
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubectl logs -n agent-broker deployment/agent-broker-openab-kiro --tail=20

# 查 PVC
sudo KUBECONFIG=/etc/rancher/k3s/k3s.yaml kubectl get pvc -n agent-broker
```

---

## 0.7.0 新功能

- **語音訊息 STT**：Discord 語音訊息自動轉文字（支援 Groq / OpenAI / Whisper）
- **圖片附件**：可直接傳圖片給 AI 分析
- **per-user 存取控制**：`allowed_users` 限制特定用戶使用
- **多 Agent 支援**：每個 agent 獨立 bot token + 頻道（kiro / claude / codex / gemini）
- **串流修復**：Discord 訊息不再被切斷
