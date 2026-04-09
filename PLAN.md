# PLAN — 專案架構與技術細節

本文件供 AI agent 或後續開發者快速掌握專案脈絡，節省探索時間。

## 專案背景

這是「陳昇瑋博士線上追思會」的虛擬場地，供參與者以 2D 像素角色形式在場地內移動、聊天、觀看影片，並與 NPC 互動。

Fork 自 [g0v/2d-online-chat](https://github.com/g0v/2d-online-chat)，原版以 Jitsi 作為即時通訊骨幹；現版已改用 [chatroom.openfun.app](https://chatroom.openfun.app) 的 WebSocket 服務，移除了所有 Jitsi 相關邏輯。

部署方式為 GitHub Pages（靜態前端）。

---

## 檔案結構

```
index.html          主頁面，登入表單 + 遊戲場景 + 所有業務邏輯（JS inline）
common.js           Canvas 工具庫：Loader（圖片載入）、Keyboard、Game loop、
                    drawGroundLayer / getDrawingHeroes / getDrawingWalls / 等渲染函式
logic-grid.js       地圖資料結構（map）、Camera、Hero 類別、Game.init / update / render
map-editor.html     管理員用地圖編輯器（另一頁面，功能獨立）
room.json           地圖資料的本地備份（離線 fallback 用）
sprite/             角色 & 地圖 Sprite 圖檔
  open_tileset.png  地圖 tileset（索引 0，室內家具牆壁等）
  moon_tileset.png  地圖 tileset（索引 1，桌椅植物等）
  school uniform 1-4/  學生角色（每套 13 男 + 18 女）
  teachers/            教師角色
loadtesting/        負載測試用 XMPP 客戶端（歷史遺留，已無實際用途）
```

---

## 前端架構

### Canvas 渲染（common.js + logic-grid.js）

- `Game` 物件為全域單例，透過 `requestAnimationFrame` 驅動 `tick → update → render`
- 渲染順序：地板層 → 物件/牆壁/角色（依 Y 座標排序，實現偽深度）
- 地圖：30 列 × 30 行，每格 32px；三個圖層：`ground`、`wall`（碰撞）、`object`（物件）
- `tile_map`（common.js）：tile 名稱 → `[tilesetIndex, col, row]` 的映射表
- `calculateWallLayer()`：從 `wall` 圖層計算出帶方向的牆壁/屋頂 tile 名稱（`wall_l`, `roof_ur` 等）

### 角色（Hero 類別，logic-grid.js）

```
hero.x / hero.y        目前像素座標（浮點）
hero.target_x / target_y  目標座標（平滑移動用）
hero.row               面向：0=下, 1=左, 2=右, 3=上
hero.col               走路動畫計時器（累積像素距離，每 50px 換一幀，共 3 幀）
hero.messages          聊天泡泡 [[text, expireTimestamp], ...]，最多 3 則
hero.image             載入好的 HTMLImageElement
```

- `me`（自身）：鍵盤控制，`Hero.SPEED = 256px/s`，有碰撞偵測
- 其他人：接收 `target_x/target_y`，以 `otherMove()` 平滑插值移動

### 位置同步節流（logic-grid.js Game.update）

- 同房間人數 < 100：每 100ms 廣播一次自身座標
- ≥ 100 人：間隔等於人數（毫秒），減少訊息量

---

## 即時通訊層（index.html inline script）

### 後端

- **WebSocket**：`wss://chatroom.openfun.app/ws`
- **房間**：固定為 `swonline`（`var room_id = 'swonline'`，index.html:726）
- 詳細協定見 [ROBOTS.md](ROBOTS.md) 或 [chatroom.openfun.app/API.md](https://chatroom.openfun.app/API.md)

### 連線函式：`connectWebSocket()`（index.html:327）

- 連線後送 `join`，帶上 `username`、`character`、初始座標
- 建立 `room` 物件，封裝 `setLocalParticipantProperty`、`sendTextMessage`、`broadcastEndpointMessage` 等介面，盡量保留原始 Jitsi API 命名風格以減少重構量

### 收到訊息後的處理（index.html:373~464）

| `msg.type` | 動作 |
|---|---|
| `joined` | 記錄 `my_user_id`、初始化其他人的 Hero 物件、載入歷史訊息 |
| `user-joined` | 建立新 Hero |
| `user-left` | 刪除 Hero |
| `say` + `payload.type=chat` | 更新聊天欄 + 角色頭上泡泡 |
| `say` + `payload.type=teleport` | 瞬間移動他人角色到 `[x, y]` |
| `say` + `payload.type=announcement` | Toast 公告 |
| `meta-updated` | 更新 `target_x/target_y`（平滑移動）和角色外觀 |

### 座標廣播

- 週期更新：`room.setLocalParticipantProperty('top', y)` / `('left', x)` → `set-meta`
- 點擊地圖瞬移：`room.broadcastEndpointMessage({type:"teleport", message:[x,y]})`

---

## NPC 與場景物件（後端 API）

- NPC / 圖片 / iframe 物件由另一個後端管理：`https://meet.jothon.online/api/rpg/`
- 端點：
  - `GET  rpg/getroom?room=swonline` — 取得地圖資料與物件列表
  - `POST rpg/addobject?room=swonline` — 新增物件
  - `POST rpg/updateobject?room=swonline&room_object_id=<id>` — 更新物件
  - `POST rpg/deleteobject?room=swonline&room_object_id=<id>` — 刪除物件
- 若後端不通，fallback 讀 `room.json`（只有地圖資料，無物件）
- `Game.objects`：以 `object_id` 為 key 的物件字典，結構為 `{type, x, y, x2, y2, data}`

### NPC 說話類型（`say_type`）

| 值 | 行為 |
|---|---|
| 1 | 不說話 |
| 2 | 玩家靠近（64px 內）才顯示 |
| 3 | 永遠顯示 |
| 4 | 輪播（定時切換） |
| 5 | 靠近 + 輪播 |

NPC 的 `say` 欄位以換行分隔多則訊息；`$people` 會被替換成目前房間人數。

---

## 部署注意事項

- 純靜態前端，可直接部署到 GitHub Pages
- 目前 branch `gh-pages` 即為部署用 branch，`master` 為開發 branch
- 不需 build 步驟，所有 JS 直接引用，無 bundler

---

## 已知限制 / 未來可改進方向

- `common.js:2` 的 `api_url` 目前指向 `https://meet.jothon.online/api/`，若該服務下線，NPC 與場景物件將無法管理（但地圖本身仍可由 `room.json` fallback 顯示）
- WebSocket 斷線後無自動重連機制，目前只顯示 alert
- 聊天歷史只在進入房間時載入一次（最多 100 筆），之後不會補載
- `map-editor.html` 有自己的一套 NPC/物件編輯邏輯，與 `index.html` 部分重複
- 角色碰撞偵測只針對 `wall` 圖層，NPC 可以重疊
