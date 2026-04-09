# 陳昇瑋博士線上追思會

這是一個 2D 像素風格的線上追思會場，用來紀念陳昇瑋博士。與會者可以選擇自己的角色，在虛擬場地中自由移動、傳訊、以及和 NPC 互動。

## 來源

本專案 fork 自 [g0v/2d-online-chat](https://github.com/g0v/2d-online-chat)。

原版採用 Jitsi 作為即時通訊後端，本版已改為採用 [chatroom.openfun.app](https://chatroom.openfun.app) 的 WebSocket API（詳見 [API 文件](https://chatroom.openfun.app/API.md)）。

## 功能

- 2D 俯視角地圖，30×30 格、每格 32px
- 支援方向鍵或點擊地圖移動角色
- 多人即時位置同步
- 聊天訊息（含歷史記錄回放）
- 表情符號快速發話
- NPC 角色（可設定靠近觸發/輪播對話）
- 場地內嵌入圖片或 iframe 物件
- 管理員可透過地圖編輯器 (`map-editor.html`) 管理場景與物件

## 素材來源

- 角色 Sprite：[Pipoya Free RPG Character Sprites 32x32](https://pipoya.itch.io/pipoya-free-rpg-character-sprites-32x32)
- 地圖 Tileset：[openpixels (silveira/openpixels)](https://github.com/silveira/openpixels)
- 方向鍵圖示：[Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Arrow_keys.svg)

## 技術

- 純前端，無需框架，使用 HTML5 Canvas + jQuery
- 即時通訊：WebSocket → `wss://chatroom.openfun.app/ws`（房間 ID：`swonline`）
- NPC 與場景物件管理（後端）：`https://meet.jothon.online/api/rpg/`

## 開發相關文件

- [PLAN.md](PLAN.md)：專案架構與技術細節，適合讓 AI agent 快速掌握脈絡
- [ROBOTS.md](ROBOTS.md)：給 AI bot 的接入說明，包含 API endpoint 與操作指令格式
