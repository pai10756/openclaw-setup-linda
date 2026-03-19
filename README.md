# OpenClaw DigitalOcean 安裝經驗與踩坑紀錄

## 伺服器環境
- DigitalOcean Droplet: Ubuntu, 1 vCPU / 1GB RAM / SGP1 (新加坡)
- OpenClaw: 2026.3.8 (3caab92)

---

## 0. 更新紀錄

### 2026-03-20: 模型切換與 Session 優化

- 原主模型 `gemini-2.5-flash` TPM 已滿，改為：
  - **主模型**: `google/gemini-3.1-flash-lite-preview`
  - **Fallback**: `google/gemini-2.0-flash`
- Session token 優化設定：
  - `compaction.mode`: `aggressive`（積極壓縮對話歷史）
  - `compaction.autoCompactThreshold`: `0.70`（context 用到 70% 就自動壓縮）
  - `compaction.maxTurns`: `20`
  - `compaction.maxTokens`: `8000`
  - `contextTokens`: `50000`（context 上限 50K tokens）
  - `maxHistoryMessages`: `30`（最多保留 30 則歷史訊息）
- 清 session 方式：`rm -rf ~/.openclaw/agents/main/sessions/*` + 重啟 gateway

### 2026-03-12: 驗證結果

- Gateway 已改為 systemd user service 常駐執行
- 內建 `web_search` 已改用 Gemini provider，不再依賴 Brave Search API key
- Telegram 上的 web search 可正常工作，但 Gemini 搜尋偶爾會遇到 `503 high demand`
- `image-gen` skill 已可在 Telegram 生成並回傳圖片
- `video-gen` skill 已建立並被 OpenClaw 識別為 ready
- Veo API 已手動驗證成功：
  - 模型：`veo-3.1-generate-preview`
  - 正確端點：`predictLongRunning`
  - 正確流程：先建立 operation，再輪詢，最後下載 MP4
- `stock-price` skill 可用，但 Yahoo Finance 仍可能出現 rate limit

### 今天確認過的關鍵差異

- Veo **不是** `:generateVideos`
- Veo 正確 REST 端點是：
  `https://generativelanguage.googleapis.com/v1beta/models/veo-3.1-generate-preview:predictLongRunning`
- Veo request body **不是**單純：
  `{ "prompt": "..." }`
- Veo request body 應使用：

```json
{
  "instances": [
    {
      "prompt": "A cute orange cat walking slowly in a sunny garden"
    }
  ],
  "parameters": {
    "aspectRatio": "16:9"
  }
}
```

---

## 1. 安裝方式

OpenClaw 是 **CLI 工具**，不是 Docker container。`docker ps` 看不到任何東西是正常的。
- 安裝後用 `openclaw` 指令操作
- 設定檔在 `~/.openclaw/openclaw.json`
- Skills 目錄: `~/.openclaw/skills/`
- Workspace: `~/.openclaw/workspace/`

---

## 2. 基本設定

### 設定檔 `~/.openclaw/openclaw.json` 結構
- `auth.profiles` — AI 模型的 API 認證 (Google API key)
- `agents.defaults.model.primary` — 預設模型 (目前: google/gemini-3.1-flash-lite-preview)
- `agents.defaults.model.fallback` — 備用模型 (目前: google/gemini-2.0-flash)
- `agents.defaults.compaction` — Session 壓縮策略
- `agents.defaults.contextTokens` — Context token 上限
- `agents.defaults.maxHistoryMessages` — 歷史訊息數上限
- `channels.telegram` — Telegram bot 設定 (botToken, dmPolicy, groupPolicy)
- `gateway` — Gateway 模式與認證 token
- `plugins.entries` — 啟用的 plugins

### Telegram 連線
- 設定在 `channels.telegram` 裡
- `dmPolicy: "pairing"`, `groupPolicy: "open"`
- `streaming: "partial"` 支援串流回覆

### 內建 web search 設定
- 內建 `web_search` 與自訂 `web-search` skill 是兩套不同東西
- 自訂 Tavily skill 不會自動覆蓋內建 `web_search`
- 內建 `web_search` 已改用 Gemini：

```json
"tools": {
  "web": {
    "search": {
      "enabled": true,
      "provider": "gemini",
      "gemini": {
        "model": "gemini-2.5-flash",
        "apiKey": "..."
      }
    }
  }
}
```

- 修改完 `openclaw.json` 後，如果目前 Gateway 還在用舊設定，要重啟實際在跑的 gateway process / service
- 舊 session 可能會殘留 Brave 設定，需要清 session 或開新對話

---

## 3. Skills 系統

### 概念
- Skills = agent 的能力/技能
- 內建 skills (openclaw-bundled): 52 個，大部分需要額外工具才能 ready
- 自建 skills (openclaw-managed): 放在 `~/.openclaw/skills/<name>/SKILL.md`

### SKILL.md 格式
```yaml
---
name: skill-name
description: "描述"
triggers:
  - "觸發關鍵字1"
  - "觸發關鍵字2"
---

# Skill 標題

說明這個 skill 做什麼。

## 使用方式

具體的 curl 指令或操作步驟，讓 agent 知道怎麼執行。
```

### 已建立的自訂 Skills

#### web-search (Tavily)
- 路徑: `~/.openclaw/skills/web-search/SKILL.md`
- 用 Tavily API 搜尋網路
- API endpoint: `https://api.tavily.com/search`
- 需要 Tavily API key

#### image-gen (Gemini 3 Pro Image)
- 路徑: `~/.openclaw/skills/image-gen/SKILL.md`
- 用 Gemini gemini-3-pro-image-preview 模型生成圖片
- API endpoint: `https://generativelanguage.googleapis.com/v1beta/models/gemini-3-pro-image-preview:generateContent`
- 需要 Google API key
- `generationConfig` 要設 `"responseModalities":["TEXT","IMAGE"]`
- 回傳 base64 編碼的 PNG

#### stock-price (Yahoo Finance)
- 路徑: `~/.openclaw/skills/stock-price/SKILL.md`
- 用 Yahoo Finance API 查即時股價
- 台股加 .TW (如 2330.TW)，美股直接用代碼

#### video-gen (Google Veo)
- 路徑: `~/.openclaw/skills/video-gen/SKILL.md`
- 用 Veo 3.1 生成短影片
- 模型：`veo-3.1-generate-preview`
- 正確 API endpoint：
  `https://generativelanguage.googleapis.com/v1beta/models/veo-3.1-generate-preview:predictLongRunning`
- 呼叫方式：
  1. `POST predictLongRunning`
  2. 取得 operation name
  3. 輪詢 operation 狀態
  4. 從 `.response.generateVideoResponse.generatedSamples[0].video.uri` 取得影片下載網址
  5. 下載成 mp4
- 手動驗證結果：可成功下載有效 MP4

### 查看 Skills 指令
```bash
openclaw skills list              # 列出所有 skills
openclaw skills list | grep ready # 只看 ready 的
openclaw skills check             # 檢查哪些 ready vs missing
openclaw skills info <name>       # 查看 skill 詳細資訊
```

---

## 4. Plugins 系統
- `openclaw plugins list` — 列出所有 plugins (通訊管道等)
- `openclaw plugins enable/disable <id>` — 啟用/停用
- Plugins 主要是 channel plugins (Telegram, Discord, WhatsApp 等)

---

## 5. 踩過的坑

### 坑 1: 以為是 Docker 部署
- `docker ps` 空的不代表沒裝好
- OpenClaw 是 CLI 工具，直接跑 `openclaw` 指令確認

### 坑 2: ClawHub Rate Limit
- `clawhub search/install/explore` 短時間跑太多次會被限速
- 錯誤訊息: `Error: Rate limit exceeded`
- **不是 DigitalOcean 的問題**，是 ClawHub API 的頻率限制
- 解法: 等 5-10 分鐘再試

### 坑 3: ClawHub install slug 格式
- `clawhub install mxfeinberg/google-search` → `Error: Invalid slug`
- `clawhub install @mxfeinberg/google-search` → 也不行
- ClawHub 的 slug 格式不直覺，搜尋出來的名字不一定能直接用
- **解法: 直接手動建立 SKILL.md，不依賴 ClawHub**

### 坑 4: Imagen API 端點
- `imagen-3.0-generate-002:predict` → 404 Not Found
- Imagen 獨立 API 可能沒開通或端點不對
- **解法: 用 `gemini-3-pro-image-preview:generateContent` 代替**
- 要在 `generationConfig` 加 `"responseModalities":["TEXT","IMAGE"]`

### 坑 5: 內建 nano-banana skill 名字被截斷
- `openclaw skills list` 顯示 `nano-banana-` (名字不完整)
- `openclaw skills info nano-banana-` → Skill not found
- 內建 bundled skill 如果 missing，不一定能直接用
- **解法: 自建 skill 更可靠**

### 坑 6: 改了設定但 Gateway 還在吃舊值
- `openclaw.json` 改完，不代表目前正在跑的 gateway process 會立即套用
- 這台一開始是舊的 `openclaw-gateway` process 還在跑，所以 `web_search` 仍然要求 Brave API key
- **解法: 停掉舊 process，改成 systemd user service，並重新啟動**

### 坑 7: 舊 session 不知道新 skill
- 新增 `video-gen` 後，舊 Telegram 對話可能仍回「沒有 video-gen skill」
- 原因是舊 session/context 沒刷新
- **解法: 清掉 `~/.openclaw/agents/main/sessions/` 後重啟 gateway，並開全新對話**

### 坑 8: image/video skill 寫死 API key 很危險
- 一開始在 `SKILL.md` 內直接寫死 Google API key
- 這樣雖然方便測試，但非常不安全，也不利後續維護
- **解法: 一律改成讀環境變數 `GEMINI_API_KEY`**

### 坑 9: Veo 端點名稱容易寫錯
- 一開始誤用：
  `.../models/veo-3.1-generate-preview:generateVideos`
- 實測正確的是：
  `.../models/veo-3.1-generate-preview:predictLongRunning`
- **解法: 用 `predictLongRunning` + operation polling**

### 坑 10: Ubuntu 只有 `python3`，不一定有 `python`
- skill 執行過程中若呼叫 `python`，可能出現：
  `python: command not found`
- **解法: 改用 `python3`，或安裝 `python-is-python3`**

### 坑 11: Session token 暴增導致 TPM 爆滿
- 長對話會快速消耗 token 限額
- 預設 compaction mode `safeguard` 太保守，不會主動壓縮
- **解法: 改用 `aggressive` mode + 設定 `autoCompactThreshold: 0.70`**
- 同時限制 `contextTokens: 50000` 和 `maxHistoryMessages: 30`
- 社群推薦參考：[OpenClaw Compaction Docs](https://docs.openclaw.ai/concepts/compaction)

### 坑 12: 主模型 TPM 用完時沒有 fallback
- 只設一個主模型，TPM 用完就整個 bot 不回覆
- **解法: 加 `model.fallback` 設定備用模型**
- 例：主模型 `gemini-3.1-flash-lite-preview`，fallback `gemini-2.0-flash`

---

## 6. 常用指令速查

```bash
# 狀態檢查
openclaw --help
openclaw status
openclaw doctor

# Skills
openclaw skills list
openclaw skills check

# Plugins
openclaw plugins list

# Gateway
openclaw gateway restart
systemctl --user restart openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager

# Config
cat ~/.openclaw/openclaw.json

# Logs
openclaw logs
journalctl --user -u openclaw-gateway --since "10 min ago" --no-pager
journalctl --user -u openclaw-gateway -f

# Session 管理
rm -rf ~/.openclaw/agents/main/sessions/*   # 清掉所有 session
systemctl --user restart openclaw-gateway.service  # 重啟 gateway 套用新設定

# ClawHub (安裝社群 skills)
npm install -g clawhub
clawhub search <keyword>
clawhub install <slug>
```

---

## 7. 待測試 / TODO
- [x] 在 Telegram 測試內建 web search (Gemini provider)
- [x] 在 Telegram 測試 image-gen skill (可正確傳送圖片)
- [x] 手動驗證 Veo API 與 mp4 下載流程
- [ ] 在 Telegram 完整測通 video-gen skill (由 bot 自動跑完整 Veo 流程)
- [ ] 建立 bus-realtime skill，待 TDX 審核通過後接正式 API
- [ ] 研究 nano-banana 完整名稱，看能否啟用內建的圖片生成 skill
- [ ] 安全性: 重新生成 Telegram bot token、gateway token、GEMINI_API_KEY
- [ ] 安全性: 將 Telegram `groupPolicy` 從 `open` 改成 `allowlist`
