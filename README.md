# OpenClaw DigitalOcean 安裝經驗與踩坑紀錄

## 伺服器環境
- DigitalOcean Droplet: Ubuntu, 1 vCPU / 1GB RAM / SGP1 (新加坡)
- OpenClaw: 2026.3.8 (3caab92)

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
- `agents.defaults.model.primary` — 預設模型 (google/gemini-2.5-flash)
- `channels.telegram` — Telegram bot 設定 (botToken, dmPolicy, groupPolicy)
- `gateway` — Gateway 模式與認證 token
- `plugins.entries` — 啟用的 plugins

### Telegram 連線
- 設定在 `channels.telegram` 裡
- `dmPolicy: "pairing"`, `groupPolicy: "open"`
- `streaming: "partial"` 支援串流回覆

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

# Config
cat ~/.openclaw/openclaw.json

# Logs
openclaw logs

# ClawHub (安裝社群 skills)
npm install -g clawhub
clawhub search <keyword>
clawhub install <slug>
```

---

## 7. 待測試 / TODO
- [ ] 在 Telegram 測試 web-search skill
- [ ] 在 Telegram 測試 image-gen skill (圖片是否能正確傳送)
- [ ] 研究 nano-banana 完整名稱，看能否啟用內建的圖片生成 skill
- [ ] 安全性: 重新生成 Telegram bot token 和 gateway token
