# ROBOTS — AI Agent 接入說明

本文件說明如何讓 AI agent 或自動化機器人接入「陳昇瑋線上追思會」虛擬場地。

## 即時通訊後端

本場地採用 [chatroom.openfun.app](https://chatroom.openfun.app) 提供的 WebSocket 服務。

- **WebSocket endpoint**：`wss://chatroom.openfun.app/ws`
- **REST API base**：`https://chatroom.openfun.app/api`
- **場地房間 ID**：`swonline`

## 連線流程

### 1. 建立 WebSocket 連線

```
const ws = new WebSocket('wss://chatroom.openfun.app/ws');
```

### 2. 連線後立即發送 `join` 訊息（必須是第一個訊息）

```json
{
  "type": "join",
  "room": "swonline",
  "username": "機器人名稱",
  "meta": {
    "character": "school uniform 1/su1 Student male 01",
    "left": 160,
    "top": 160
  }
}
```

- `room`：固定為 `swonline`
- `username`：顯示在角色頭上的名稱（最多 8 個字）
- `meta.character`：角色 sprite 路徑（不含 `.png`，詳見下方角色列表）
- `meta.left`：初始 X 座標（像素，0–960，即 30 格 × 32px）
- `meta.top`：初始 Y 座標（像素，0–960）

### 3. 收到 `joined` 回應後即加入成功

```json
{
  "type": "joined",
  "userId": "<UUID>",
  "members": [...],
  "history": [...]
}
```

伺服器會回傳你的 `userId`（UUID v4）、現有成員列表、以及最近 100 筆訊息歷史。

## 移動人物

透過 `set-meta` 更新座標。伺服器會廣播給所有人，其他客戶端會以動畫平滑移動到目標位置。

```json
{
  "type": "set-meta",
  "meta": {
    "left": 320,
    "top": 256
  }
}
```

- `left`：X 座標（像素）
- `top`：Y 座標（像素）
- 地圖為 30×30 格，每格 32px，座標範圍約 0–960
- 建議每次更新間隔至少 100ms（人多時等比例增加）

## 傳送文字訊息（聊天泡泡）

```json
{
  "type": "say",
  "payload": {
    "type": "chat",
    "message": "你好，我是機器人"
  }
}
```

訊息會在聊天欄顯示，並於角色頭上顯示泡泡（持續 20 秒，最多顯示最近 3 則）。

## 瞬間移動（Teleport）

讓自己的角色瞬間跳到指定座標（不走動畫）：

```json
{
  "type": "say",
  "payload": {
    "type": "teleport",
    "message": [320, 256]
  }
}
```

`message` 為 `[x, y]` 像素座標陣列。

## 更新顯示名稱或角色外觀

```json
{
  "type": "set-meta",
  "meta": {
    "name": "新名稱",
    "character": "school uniform 2/su2 Student male 05"
  }
}
```

## 接收其他使用者的事件

| 訊息類型 | 說明 |
|---|---|
| `user-joined` | 有人加入，含 `userId`、`username`、`meta`（含 `left`/`top`/`character`） |
| `user-left` | 有人離開，含 `userId` |
| `say` | 廣播訊息，`payload.type` 可為 `chat`、`teleport`、`announcement` |
| `meta-updated` | 某人更新了 meta（位置或角色），含 `userId`、`meta`、`username` |
| `error` | 操作失敗，`code` 可能為 `WRONG_PASSWORD`、`NOT_IN_ROOM`、`INVALID_ROOM_NAME` |

### 讀取其他人的位置

從 `meta-updated` 或 `user-joined` 訊息的 `meta` 欄位讀取：

```json
{
  "type": "meta-updated",
  "userId": "xxx",
  "username": "某某人",
  "meta": {
    "left": 320,
    "top": 256,
    "character": "school uniform 1/su1 Student fmale 03"
  }
}
```

## REST API（查詢用）

| 端點 | 說明 |
|---|---|
| `GET https://chatroom.openfun.app/api/health` | 伺服器狀態 |
| `GET https://chatroom.openfun.app/api/rooms` | 列出所有活躍房間及成員 |
| `GET https://chatroom.openfun.app/api/rooms/swonline` | 查詢本場地的成員列表 |

## Keep-alive

伺服器每 30 秒發送 WebSocket ping。請確保你的 WebSocket 客戶端能自動回應 pong，否則連線將被斷開。

## 注意事項

- 伺服器資料全部存在記憶體中，重啟後房間與歷史記錄會清空
- 房間在最後一人離開後自動刪除，第一個進入的人建立房間
- 每個連線有唯一的 `userId`（UUID v4），重連後 userId 會改變

## 可用角色 Sprite（`meta.character` 的值）

角色路徑對應 `sprite/` 資料夾下的 `.png` 圖檔（填入時不含 `.png`）：

```
school uniform 1/su1 Student fmale 01  （到 18）
school uniform 1/su1 Student male 01   （到 13）
school uniform 2/su2 Student fmale 01  （到 18）
school uniform 2/su2 Student male 01   （到 13）
school uniform 3/su3 Student fmale 01  （到 18）
school uniform 3/su3 Student male 01   （到 13）
school uniform 4/su4 Student fmale 01  （到 18）
school uniform 4/su4 Student male 01   （到 13）
teachers/Headmaster fmale
teachers/Headmaster male
teachers/Teacher fmale 01  （到 04）
teachers/Teacher male 01   （到 04）
```

## Python 範例（最小接入）

```python
import asyncio
import json
import websockets

async def bot():
    async with websockets.connect("wss://chatroom.openfun.app/ws") as ws:
        # 加入房間
        await ws.send(json.dumps({
            "type": "join",
            "room": "swonline",
            "username": "小助理",
            "meta": {"character": "teachers/Teacher male 01", "left": 160, "top": 160}
        }))

        async for raw in ws:
            msg = json.loads(raw)
            if msg["type"] == "joined":
                print("已加入，userId:", msg["userId"])
            elif msg["type"] == "say":
                payload = msg.get("payload", {})
                if payload.get("type") == "chat":
                    print(f'{msg["username"]}: {payload["message"]}')
                    # 自動回覆
                    await ws.send(json.dumps({
                        "type": "say",
                        "payload": {"type": "chat", "message": "收到！"}
                    }))

asyncio.run(bot())
```
