# 群組裡沒有 Loading，卡片也放錯按鈕：一個 LINE Bot 上線後才出現的兩道題

上一篇，我把 LINE Cafe Bot 邀進朋友群組，讓大家可以從附近咖啡廳中加入候選、各投一票，最後再一起決定去哪間。

程式完成、測試通過後，我真的打開 LINE 試了一次。

我在群組傳送目前位置，然後看著聊天室等了一下。

**什麼都沒有。**

過一會兒，咖啡廳推薦終於出現了，但卡片上的按鈕是：

```text
安排喝咖啡時間
記錄這次造訪
加入想去清單
```

我想做的「加入群組候選」和「投票」反而完全找不到。

這次最值得記下來的，不是我一次就把功能做對，而是兩個問題都必須真的進到群組裡，才看得出來。

完整程式碼：
https://github.com/zonawang/line-cafe-group-planner

## 第一個問題：為什麼群組沒有 Loading 動畫？

原本 Bot 收到位置後，會呼叫 LINE 的 Loading Animation API：

```ts
await lineClient.showLoadingAnimation({
  chatId: targetId,
  loadingSeconds: 60
});
```

在一對一聊天室中，`targetId` 是使用者的 user ID，所以畫面會顯示 Bot 正在處理。

進到群組後，`targetId` 變成 group ID。程式一樣執行這段，畫面卻沒有動畫。

我和 Codex 往 LINE SDK 與官方說明查，才確認 Loading Animation 的適用範圍是：

> 使用者與 LINE Official Account 之間的一對一聊天室。

它不是群組聊天室的 Loading 元件，而且 request 裡的 `chatId` 要放目標使用者的 user ID，不是 group ID。

原本程式又剛好用 `try...catch` 包住這段 API：

```text
群組呼叫 Loading API
        ↓
LINE 拒絕 group ID
        ↓
錯誤被記進 log
        ↓
咖啡廳搜尋仍然繼續
```

所以功能沒有整個壞掉，使用者看到的只是「安靜很久，然後結果突然出現」。

## API 不支援，就改用聊天室本來就有的東西

這個問題不是換一個參數就能讓動畫出現在群組。

最後的做法是依照 webhook source 分流：

```text
一對一聊天室 → 使用 Loading Animation
LINE 群組     → 立即回覆一則文字狀態
```

群組收到位置後，現在會先看到：

> ☕ 收到位置！正在幫大家找附近的咖啡廳，請稍等一下⋯
>
> 找到後可以把喜歡的店加入群組候選，再一起投票。

這不是動畫，但它做到了更重要的兩件事：

1. 告訴大家 Bot 確實收到位置了。
2. 順便預告搜尋完成後要做什麼。

有時候最適合的替代方案，不是想辦法模仿原本的 UI，而是用平台確實支援的方式，把狀態說清楚。

## 第二個問題：群組為什麼還是出現個人版卡片？

這個問題後來發現有兩層。

第一層是原本的程式設計。

最初版本假設使用者一定會先輸入：

```text
一起選咖啡廳
```

這個指令會建立一輪投票並取得 `groupPlanId`。之後傳送位置時，只要有這個 ID，Bot 就顯示群組版卡片。

可以把判斷簡化成：

```text
有 groupPlanId  → 加入群組候選
沒有 groupPlanId → 安排時間／記錄造訪／個人收藏
```

問題是，真實使用時我沒有先想這麼多。我進到群組，看到可以傳位置，就直接按了。

程式沒有壞，它只是忠實走進「沒有 groupPlanId」的分支。但對使用者來說，這就是找不到群組功能。

## 不該要求使用者記得正確的開場白

我和 Codex 最後把規則改成：

> 只要位置訊息來自 LINE 群組，就自動建立或沿用目前的群組投票。

因此兩種操作現在都成立：

```text
輸入「一起選咖啡廳」→ 傳位置
直接在群組傳位置
```

如果群組已經有進行中的投票，就沿用原本候選和票數，不會因為另一個人又傳位置而全部重置。

如果上一輪已截止或過期，才會建立新的 plan ID。

這次我學到的是：文字指令可以當入口，但不該變成使用者必須記住的通關密語。

## 還有一層：GitHub 有新功能，不代表 LINE 正在跑新功能

排除程式分支後，我們又檢查了 Cloud Run。

結果發現，當時根本還沒有名為：

```text
line-cafe-group-planner
```

的線上服務。

LINE webhook 仍然指向上一站：

```text
line-cafe-menu-recommender
```

也就是說，我在 GitHub 上看到的是新的群組程式，LINE 實際呼叫的卻還是舊服務。

舊服務不知道什麼是 `groupPlanId`，當然只會回傳原本的個人功能卡片。

這是一個很容易忽略的差別：

```text
程式碼已 push ≠ Cloud Run 已部署 ≠ LINE webhook 已切換
```

三件事必須分開確認。

## 我把卡片也一起重新整理

即使把群組按鈕加回來，如果一張卡仍然塞著五個功能，使用者還是要花時間找下一步。

所以群組模式的推薦卡最後只留下：

```text
加入群組候選
先看地圖資訊
```

「加入群組候選」移到第一個按鈕，並使用主要按鈕樣式。

在推薦卡片出現前，Bot 還會先說明：

```text
1. 左右滑動比較推薦店家
2. 在喜歡的店卡片上點「加入群組候選」
3. 加好候選後，點「查看並投票」
```

卡片下方也固定放上「查看並投票」。

這樣不只功能存在，使用者也看得出來接下來該做什麼。

## 投票按鈕為什麼不直接出現在第一批搜尋卡片？

搜尋結果還不是正式候選。

群組可能看到五間推薦，但只想留下其中兩、三間比較。如果每一張搜尋卡都直接計票，大家會不知道哪些店已經進入共同名單。

所以流程刻意分成兩步：

```text
搜尋卡片：加入群組候選
投票卡片：投這間
```

加入第一間候選後，Bot 會提供「查看並投票」。進到投票卡片後，每間店才會顯示目前票數與「投這間」。

這多一次按鈕，卻把「提出選項」和「表達選擇」分得更清楚。

## Codex 不只改程式，也一起確認我到底測到哪一版

這次如果只盯著程式碼，很容易一直問：「群組按鈕明明寫了，為什麼 LINE 沒有？」

Codex 幫我從四個層次逐一確認：

```text
平台層：Loading Animation 是否支援群組
流程層：直接傳位置時，有沒有建立 groupPlanId
介面層：群組卡片是否把主要動作放在前面
部署層：LINE webhook 實際指向哪個 Cloud Run service
```

它也沒有把所有問題都歸因於「可能是快取」。

我們從 LINE SDK 的型別說明確認 Loading 只限一對一；從程式分支找到沒有 plan ID 時的個人卡片；再從 Google Cloud 查出新服務尚未存在，最後讀取 LINE webhook 的現行網址。

對我來說，這次 Codex 最有價值的地方不是快速多寫一個按鈕，而是把「我在 LINE 看到的不對」一路追到可以驗證的原因。

## 這次部署，我沒有直接蓋掉上一站

修正完成後，我們建立新的 Cloud Run 服務：

```text
line-cafe-group-planner
```

也建立獨立的 runtime service account，只授予功能需要的角色：

- Vertex AI 使用權限。
- Firestore 讀寫權限。
- Cloud Tasks 建立任務權限。
- Service Usage 權限。

部署時沿用 512 MiB、1 CPU 與 `--no-cpu-throttling`，並加入群組投票使用的 Firestore collection。

接著依序驗證：

1. 66 項測試全數通過。
2. Cloud Run `/health` 回傳成功。
3. 使用真正的 LINE channel secret 產生簽章，新 webhook 回傳 `200`。
4. 回訪提醒 callback 已改到新服務。
5. LINE 官方 Webhook Verify 回傳 `200`。
6. webhook 狀態為啟用後，才正式切換網址。

如果 LINE Verify 失敗，切換腳本會自動改回舊 endpoint。上一站的 Cloud Run 服務也沒有刪除，因此仍然保有回復路徑。

最後上線的是：

```text
line-cafe-group-planner-00002-lcj
```

## 測試通過，不等於使用體驗已經通過

這次 66 項自動測試涵蓋：

- 群組 action 的解析。
- postback 長度限制。
- 候選去重與票數計算。
- 單一最高票與平手。
- Flex Message 卡片與按鈕。
- 群組搜尋中的文字狀態。
- 舊輪次與錯誤操作的保護。

它們可以證明程式按照規則運作，卻無法代替我真的拿起手機，看一眼「下一步到底明不明顯」。

這兩種測試解決的是不同問題：

```text
自動測試：程式有沒有照規則做
真實操作：人知不知道接下來怎麼做
```

少了任何一邊，都可能得到一個技術上正確、實際上很難使用的 Bot。

## 這次留下來的三個提醒

第一，平台元件有自己的適用範圍。一對一聊天室能用的 Loading Animation，不一定能原封不動搬到群組。

第二，不要把理想操作順序當成使用者一定知道的事。如果直接傳位置很合理，程式就應該接得住。

第三，除錯時要確認線上實際執行的版本。GitHub、Cloud Run 和 LINE webhook 是三個不同狀態，不能只看其中一個。

我原本只是想讓朋友不要再一直說「我都可以」。最後做完的不只是群組投票，也多了一套更誠實的檢查方式：

> 功能寫進程式只是第一步；使用者找得到、線上真的跑到，而且平台確實支援，才算完成。

## 完整程式碼與官方文件

GitHub：
https://github.com/zonawang/line-cafe-group-planner

LINE Messaging API — Display a loading animation：
https://developers.line.biz/en/reference/messaging-api/#display-a-loading-indicator

LINE Messaging API — Group chats：
https://developers.line.biz/en/docs/messaging-api/group-chats/

LINE Messaging API — Get webhook endpoint information：
https://developers.line.biz/en/reference/messaging-api/#get-webhook-endpoint-information

LINE Messaging API — Test webhook endpoint：
https://developers.line.biz/en/reference/messaging-api/#test-webhook-endpoint
