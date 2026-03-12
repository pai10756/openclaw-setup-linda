---
name: bus-realtime
description: "查詢台灣公車即時到站時間。當使用者輸入站名與公車號碼，想知道還有幾分鐘到站時使用。優先支援台北市與新北市。"
---

# Bus Realtime Skill

使用台灣交通部 TDX 官方公車即時資料，根據「站名 + 公車號碼」查詢預估到站時間。

## Requirements

- 必須先申請 TDX API 金鑰
- 必須設定環境變數：
  - `TDX_CLIENT_ID`
  - `TDX_CLIENT_SECRET`
- 系統需可使用：
  - `curl`
  - `jq`

如果缺少任一環境變數，直接回覆：
`目前尚未設定 TDX_CLIENT_ID / TDX_CLIENT_SECRET，所以無法查詢公車動態。`

## Scope

- 先優先查詢 `Taipei`、`NewTaipei`
- 使用者若未指定縣市，先在台北市查，找不到再查新北市
- 輸入通常是：
  - `內湖高中 287`
  - `三總站 紅2`
  - `捷運西湖站 645`

## Goal

回覆長輩看得懂的結果，只說重點：

- 路線
- 站名
- 方向
- 還有幾分鐘到站 / 進站中 / 尚未發車 / 今日未營運

不要回一大串原始 JSON。

## TDX Authentication

先用 Client Credentials 取得 access token：

```bash
TOKEN_JSON="$(mktemp)"

curl -s --request POST \
  --url 'https://tdx.transportdata.tw/auth/realms/TDXConnect/protocol/openid-connect/token' \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data grant_type=client_credentials \
  --data client_id="$TDX_CLIENT_ID" \
  --data client_secret="$TDX_CLIENT_SECRET" > "$TOKEN_JSON"

ACCESS_TOKEN="$(jq -r '.access_token // empty' "$TOKEN_JSON")"
```

如果 `ACCESS_TOKEN` 為空，回覆：
`目前無法連線到公車資料服務，請稍後再試。`

## Query Strategy

對每個候選城市依序執行：

1. 先查該城市指定路線的預估到站資料：

```bash
CITY="Taipei"
ROUTE="287"
ETA_JSON="$(mktemp)"

curl -s --request GET \
  --url "https://tdx.transportdata.tw/api/basic/v2/Bus/EstimatedTimeOfArrival/City/${CITY}/${ROUTE}?\$format=JSON" \
  --header "authorization: Bearer ${ACCESS_TOKEN}" \
  --header 'accept-encoding: gzip' > "$ETA_JSON"
```

2. 用站名比對 `StopName/Zh_tw`
3. 若有多筆，優先回：
   - `EstimateTime` 最小者
   - 若沒有 `EstimateTime`，看 `StopStatus`

## Matching Rules

- 站名採「包含比對」優先，例如使用者輸入 `三總`，可匹配 `三軍總醫院`
- 若同一路線同站名出現雙向資料，最多回 2 筆，並標示方向
- 方向欄位優先使用：
  - `DestinationStopNameZh`
  - 若無，退回 `Direction`（0/1）

## Status Mapping

依下列規則轉成人話：

- `EstimateTime <= 30` 秒：`進站中`
- `EstimateTime > 30` 秒：換算為分鐘，無條件進位
- `StopStatus = 0`：正常顯示預估時間
- `StopStatus = 1`：`尚未發車`
- `StopStatus = 2`：`交管不停靠`
- `StopStatus = 3`：`末班車已過`
- `StopStatus = 4`：`今日未營運`

## Response Style

回覆使用繁體中文，短句、口語、長輩看得懂。

好格式：

`287 在內湖高中站，往東湖方向，約 3 分鐘到站。`

`紅2 在三軍總醫院站，往捷運圓山站方向，進站中。`

若找到兩個方向：

`287 在內湖高中站：`
`往東湖方向，約 3 分鐘到站。`
`往公館方向，約 12 分鐘到站。`

若找不到：

`我目前找不到「站名 + 公車號碼」的對應資料，請再告訴我更完整的站名。`

## Notes

- 不要改用一般網頁搜尋當主要來源，優先用 TDX 官方資料
- 不要一次列太多筆，最多 2 筆
- 不要把英文城市代碼直接回給使用者
- 若台北市與新北市都找不到，再請使用者補充站名
