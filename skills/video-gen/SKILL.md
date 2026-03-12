---
name: video-gen
description: "生成短影片或動畫。當使用者要求做動畫、短片、動態影片、Veo 影片時使用。優先使用 Google Veo 類影片生成 API。"
---

# Video Generation Skill

使用 Google 影片生成 API 建立短影片。這個 skill 適合處理：

- 動畫短片
- 商品展示短影片
- 場景動態影片
- 文字轉影片

## Requirements

- 必須先設定 `GEMINI_API_KEY`
- 系統需可使用：
  - `curl`
  - `jq`
- 若缺少 `GEMINI_API_KEY`，直接回覆：
  `目前尚未設定 GEMINI_API_KEY，所以無法生成影片。`

## Usage Policy

當使用者要求「動畫」、「短影片」、「Veo」、「影片生成」、「做一段會動的畫面」時，優先使用此 skill。

不要只回文字描述。
不要假裝影片已生成。
如果 API 僅回傳 operation/job，必須繼續輪詢直到完成或超時。

## Prompting

先將使用者需求整理成清楚的英文 prompt，內容盡量包含：

- subject
- scene
- camera movement
- style
- duration
- aspect ratio

例如：

`A cute orange cat walking slowly in a sunny garden, cinematic lighting, gentle camera pan, soft animation style, short video`

## API Pattern

影片生成通常是非同步工作。請使用以下流程：

1. 提交影片生成請求
2. 取得 operation / job id
3. 輪詢 operation 狀態
4. 完成後擷取影片下載 URL 或 inline 資料
5. 若能下載影片，存成本地檔案並回傳給使用者

## Request Template

注意：實際模型名稱與端點可能隨版本調整。如果失敗，先把完整 API 回應存起來並回報。

先建立請求：

```bash
PROMPT="PROMPT_HERE"
REQ_JSON="$(mktemp)"
RESP_JSON="$(mktemp)"

cat > "$REQ_JSON" <<EOF
{
  "prompt": "$PROMPT"
}
EOF
```

送出請求：

```bash
curl -s \
  -H "Content-Type: application/json" \
  -H "x-goog-api-key: $GEMINI_API_KEY" \
  -d @"$REQ_JSON" \
  "https://generativelanguage.googleapis.com/v1beta/models/veo:generateVideos" \
  > "$RESP_JSON"
```

## Operation Handling

回應後：

- 若有 `.name` 或 operation id，繼續輪詢
- 若有 `.error`，直接回報錯誤
- 若模型不存在、權限不足、API 未開通，明確告知使用者

範例輪詢：

```bash
OP_NAME="$(jq -r '.name // empty' "$RESP_JSON")"
```

如果 `OP_NAME` 為空，先輸出 API 回應：

```bash
cat "$RESP_JSON"
```

若有 operation id，輪詢最多 30 次，每次等 10 秒：

```bash
STATUS_JSON="$(mktemp)"
for i in $(seq 1 30); do
  curl -s \
    -H "x-goog-api-key: $GEMINI_API_KEY" \
    "https://generativelanguage.googleapis.com/v1beta/${OP_NAME}" \
    > "$STATUS_JSON"

  DONE="$(jq -r '.done // false' "$STATUS_JSON")"
  if [ "$DONE" = "true" ]; then
    break
  fi
  sleep 10
done
```

## Result Handling

完成後優先找：

- 影片下載 URL
- 可直接下載的檔案欄位
- 任何 `video` / `uri` / `downloadUri` / `fileUri` 之類欄位

若找到影片 URL，下載到本地檔案：

```bash
VIDEO_URL="$(jq -r '.. | .uri? // empty' "$STATUS_JSON" | head -n 1)"
OUT="/tmp/openclaw-video-$(date +%s).mp4"

if [ -n "$VIDEO_URL" ]; then
  curl -L -s "$VIDEO_URL" -o "$OUT"
fi
```

下載後檢查：

```bash
test -s "$OUT"
```

- 若成功，將影片檔回傳給使用者
- 若失敗，回覆：
  `影片生成已完成，但目前無法正確下載影片檔。`

## Failure Handling

如果 API 回傳下列情況，請直接明講，不要模糊帶過：

- `403` / `PERMISSION_DENIED`
  - `目前這組 Google API key 沒有影片生成模型權限。`
- `404` / `model not found`
  - `目前設定的影片模型名稱或端點不可用。`
- `429`
  - `目前請求過多，請稍後再試。`
- `500` / `503`
  - `影片生成服務暫時忙碌，請稍後再試。`

## Response Style

回覆使用繁體中文，簡短明確。

成功時：

`影片已生成，這是為您建立的短片。`

失敗時：

`目前影片生成失敗，原因是 API 權限不足。`

## Notes

- 不要在 skill 檔中寫死 API key
- 不要只回傳原始 JSON 給使用者
- 若 API 規格改動，先保留完整回應以便除錯
- 這是影片生成 skill，不要拿來做純圖片生成
