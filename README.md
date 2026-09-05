# LINE Cafe Group Planner

把 LINE Cafe Bot 加進群組後，大家可以一起收集咖啡廳候選、各投一票，再由發起人截止並公布結果。

這個版本延續 [`line-cafe-menu-recommender`](https://github.com/zonawang/line-cafe-menu-recommender) 的附近搜尋、偏好、行程、回訪、咖啡足跡、想去清單與拍菜單推薦功能。

## 群組使用流程

```text
群組輸入「一起選咖啡廳」
        ↓
發起人傳送聚會地點附近的位置
        ↓
Google Maps Grounding 推薦附近咖啡廳
        ↓
從推薦卡加入最多 5 間群組候選
        ↓
每位群組成員投 1 票，也可以改票
        ↓
發起人截止，Bot 公布最高票或平手結果
```

查看最新票數可輸入：

```text
查看群組投票
```

## 群組功能

- 僅接受 LINE `group` webhook source，不會把私人聊天室誤當成群組。
- Bot 被邀進群組時，會主動說明「一起選咖啡廳」入口。
- 每個群組同時保留一輪投票，新一輪開始時會產生新的 plan ID。
- 推薦卡片可直接加入群組候選，依 Google Maps URI 去除重複店家。
- 每輪最多 5 間候選，效期 24 小時。
- 每位 LINE 使用者只有一張有效票，再次投票會改票而不是累加。
- 所有成員都能即時查看票數；只有發起人能截止。
- 平手時列出所有最高票店家，不擅自挑選贏家。
- 舊輪次、跨群組、已截止與過期的 postback 都會被拒絕。

群組、候選與票數保存在 Firestore。儲存的是 LINE user ID、group ID、Google Maps 店名與網址，不會額外取得或保存成員的顯示名稱。

## 既有功能

- Gemini + Google Maps Grounding 附近咖啡廳推薦。
- 個人偏好、換一批與更適合工作。
- Datetime Picker、Google Calendar 與造訪後主動回訪。
- 1～5 分、體驗標籤與咖啡足跡。
- 想去清單的新增、去重、查看、安排時間與移除。
- 拍攝菜單後，以 Gemini 多模態推薦最多三杯可見飲品。
- LINE Rich Menu。

## LINE 官方帳號設定

除了既有的 Messaging API webhook，還要在 LINE Developers Console 的 Messaging API 設定中開啟：

```text
Allow bot to join group chats
```

同一個 LINE 群組一次只能加入一個 LINE 官方帳號。設定開啟後，把 Bot 邀進測試群組，再輸入「一起選咖啡廳」。

## 本機設定

需求：Node.js 20 以上、LINE Messaging API channel，以及已啟用 Vertex AI、Firestore 與 Cloud Tasks 的 Google Cloud 專案。

```bash
cp .env.example .env
npm install
npm run dev
```

本機呼叫 Google Cloud 服務前，先建立 Application Default Credentials：

```bash
gcloud auth application-default login
```

群組功能新增的環境變數：

```env
FIRESTORE_GROUP_PLANS_COLLECTION=cafe-group-plans
```

完整設定請參考 [`.env.example`](.env.example)。

## 驗證

```bash
npm run typecheck
npm test
```

目前共有 65 項測試，CI 會在 push 與 pull request 執行相同檢查。

## Cloud Run 部署

webhook 會先回傳 `200`，再在背景處理圖片與部分推送流程，因此部署時保留 `--no-cpu-throttling`：

```bash
gcloud run deploy line-cafe-group-planner \
  --source . \
  --region asia-east1 \
  --allow-unauthenticated \
  --no-cpu-throttling \
  --service-account line-cafe-group-planner@YOUR_PROJECT.iam.gserviceaccount.com \
  --env-vars-file cloud-run-env.yaml
```

確認服務正常後，把 LINE webhook 指向：

```text
https://YOUR_SERVICE_URL/webhook
```

健康檢查：`GET /health`

## 已知限制

- 目前每個群組同時只進行一輪投票。
- 成員無法移除候選店；候選選錯時可重新開始一輪。
- 截止時間不是排程自動觸發，而是由發起人按下「截止並公布結果」。
- LINE 不提供群組 Rich Menu 專屬入口，因此以文字指令「一起選咖啡廳」開始。

## 官方文件

- [LINE Messaging API：群組與多人聊天室](https://developers.line.biz/en/docs/messaging-api/group-chats/)
- [LINE Messaging API：Webhook event objects](https://developers.line.biz/en/reference/messaging-api/#webhook-event-objects)
- [LINE Messaging API：Postback action](https://developers.line.biz/en/reference/messaging-api/#postback-action)
- [LINE Messaging API：Flex Message](https://developers.line.biz/en/docs/messaging-api/using-flex-messages/)
