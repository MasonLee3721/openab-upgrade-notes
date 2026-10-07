# OpenAB 運維與部署紀錄存檔 (OpenAB Upgrade & Ops Notes)

本儲存庫記錄 OpenAB 在 k3s 叢集上的重大版本升級、多 Agent 擴充、架構除錯與維運筆記。

---

## 📑 筆記目錄

1. **[2026-10-07 新增第 9 位 Agent「陳近南」部署與排錯實錄](./2026-10-07-chenjinnan-pi-agent-deployment.md)**
   - 整合 `@earendil-works/pi-coding-agent` (GPT-5.5) + `@ccgv2/pi-acp`
   - 單節點記憶體調度瓶頸排查 (`requests.memory: 128Mi`)
   - 遠端 Kubernetes ChatGPT OAuth 認證流程與回呼攔截
   - `pi-acp` 遺失 `OPENAI_API_KEY` 造成 0 token 空回覆之 Root Cause 與 Wrapper 橋接架構
   - 鹿鼎記 9 人全員互信 `trustedBotIds` 與「先生 / Mason兄」人設打磨

2. **[2026-04-12 OpenAB Helm 升級紀錄 (0.3.3 → 0.7.0)](./openab-upgrade-notes.md)**
   - Helm Repository 遷移與 Values 結構調整
   - STT 語音轉文字、多 Agent 獨立頻道支援
