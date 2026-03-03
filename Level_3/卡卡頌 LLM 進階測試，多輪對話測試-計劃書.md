# Project Akka — 卡卡頌 LLM 測試計劃 (Level 3)

多輪對話場景測試，模擬真實桌遊店情境。  
中文與英文雙語對照，測試本地與雲端模型的規則理解與上下文追蹤能力。

---

## 基本資訊

| 項目 | 內容 |
|------|------|
| 測試層級 | Level 3 — 多輪對話 |
| 場景數量 | 10 |
| 測試語言 | 中文（ZH）+ 英文（EN）|
| 推論引擎 | Ollama 0.16.3 |
| API 端點 | `http://192.168.50.33:11434/api/chat` |
| 評分方式 | 純記錄，人工評估 |

---

## 測試模型

| 模型 | Thinking | 備註 |
|------|:--------:|------|
| `qwen3:0.6b` | ✅ | |
| `qwen3:1.7b` | ✅ | |
| `qwen3:4b` | ✅ | |
| `qwen3:4b-instruct` | ❌ | 關閉 thinking |
| `qwen3:8b` | ✅ | |
| `qwen3:14b` | ✅ | |
| `qwen3:30b-instruct` | ❌ | 關閉 thinking |
| `gemma3:270m` | — | 基準參照 |
| `gemma3:1b` | — | |
| `gemma3:4b` | — | |
| `gemma3:12b` | — | 本地部署候選 |
| `gemma3:27b-cloud` | — | 雲端 fallback 參照 |

---

## System Prompt

### 中文版

```
你是一個桌遊咖啡廳的服務員，負責用輕鬆、口語化的方式回答客人關於卡卡頌的問題。

回答規則：
- 用朋友聊天的語氣，不要太正式。
- 字數限制：中文回答控制在 100 字內。
- 可以多用「喔」、「啦」、「嘛」等語助詞，讓對話更自然。
- 如果不確定，直接說不確定，不要瞎猜。

卡卡頌完整規則：
1. 放置板塊：每回合抽一張板塊，必須接在已有板塊旁邊，道路接道路、城市接城市、草原接草原。
2. 放置米寶：「當前放板塊的玩家」放完板塊後，只能在「該片板塊」上選擇一個區域放置「一個」米寶。
   其他玩家不能在別人的回合放米寶。
3. 米寶種類：騎士放城市、強盜放道路、修士放修道院、農夫躺在草原。
4. 計分時機：
   - 道路完成：每個板塊 1 分。
   - 城市完成：每個板塊 2 分（每個盾牌標記再加 2 分）。
   - 修道院完成：修道院本身搭配周圍 8 片板塊填滿，共得 9 分。
   - 遊戲結束時未完成建築：每片板塊 1 分、每個盾牌標記 1 分。
   - 農夫在遊戲結束時計分：農夫所在草原每連接到一個「已完成」的城市得 3 分。
     未完成的城市不計入農夫分數。
5. 多玩家共佔規則：
   - 建築完成時，若多個玩家在該建築放了相同數量的米寶，所有人都得全額分數（不平分）。
   - 若米寶數量不同，只有米寶最多的玩家得分。
6. 米寶回收：道路、城市、修道院完成後，該區域米寶回到玩家手上。農夫不會中途回收。
```

### 英文版

```
You are a server at a board game café, responsible for answering customers' questions about
Carcassonne in a relaxed, conversational manner.

Response Rules:
- Use a casual, friendly tone like chatting with a friend.
- Word limit: keep responses under 200 words.
- You can use conversational fillers like "well," "you know," "so," etc.
- If you're uncertain, just say you're not sure — don't guess.

Complete Carcassonne Rules:
1. Tile Placement: Each turn, draw one tile and place it adjacent to existing tiles.
   Roads connect to roads, cities to cities, fields to fields.
2. Meeple Placement: After placing a tile, only the CURRENT player may place exactly ONE meeple
   on ONE area of THAT tile. Other players cannot place meeples during another player's turn.
3. Meeple Types: Knight goes in city, Robber goes on road, Monk goes in monastery, Farmer lies in field.
4. Scoring:
   - Completed road: 1 point per tile.
   - Completed city: 2 points per tile (plus 2 points per shield pennant).
   - Completed monastery: 9 points (monastery tile + all 8 surrounding spaces filled).
   - Incomplete features at game end: 1 point per tile, 1 point per shield pennant.
   - Farmers score at game end: 3 points per COMPLETED city connected to their field.
     Incomplete cities do NOT count toward farmer scoring.
5. Shared Scoring: When a feature completes and multiple players have EQUAL meeple counts there,
   ALL tied players receive the FULL score — the score is NOT split.
   If meeple counts differ, only the player(s) with the most meeples score.
6. Meeple Return: Meeples return to hand when roads, cities, or monasteries complete.
   Farmers are NEVER returned during the game.
```

---

## 評分維度

每個場景滿分 100 分，分四個維度：

| 維度 | 配分 | 說明 |
|------|:----:|------|
| 規則正確性 | 40 | 最終答案的規則與計算是否正確 |
| 上下文追蹤 | 20 | 是否正確記住並使用多輪累積的資訊（含更正） |
| 語氣自然度 | 20 | 是否符合店員語氣，像真人對話 |
| 字數控制 | 20 | 中文 ≤ 100 字 / 英文 ≤ 200 words |

---

## 測試場景

### 場景 1 — 碎片資訊累積 → 城市計分

**測試目的：** 客人邊玩邊說，資訊分多輪給出，測試模型的上下文記憶與更新能力

| 輪次 | 中文 | English |
|------|------|---------|
| 1 | 我剛放了一片城市板塊。 | I just placed a city tile. |
| 2 | 啊對，我之前有放騎士在這個城市，現在總共 4 片了。 | Oh right — I placed a knight in this city earlier. Now it has 4 tiles total. |
| 3 | 城市剛被圍起來了，我得幾分？ | The city just got completed. How many points do I get? |

**正確答案：** 8 分（4 片 × 2 分）。騎士回到手上。

**評分要點：**
- 記住「騎士在城市」（第二輪才補充，不是第一輪）
- 記住「4 片」（第二輪更新後的數字）
- 計算正確（8 分）
- 提醒米寶回收

---

### 場景 2 — 客人說錯再更正 → 盾牌加分計算

**測試目的：** 客人中途修正自己的資訊，測試模型能否正確更新記憶

| 輪次 | 中文 | English |
|------|------|---------|
| 1 | 我完成了一個城市，3 片板塊，有 2 個盾牌。 | I completed a city — 3 tiles, with 2 shields. |
| 2 | 不對不對，我數錯了，是 5 片板塊。 | Wait, I miscounted — it's actually 5 tiles. |
| 3 | 我的騎士在上面，可以得幾分？ | My knight is on it — how many points do I get? |

**正確答案：** 14 分（5×2=10 + 2×2=4）。使用更正後的 5 片。騎士回到手上。

**評分要點：**
- 接受更正，使用 5 片（不是原來的 3 片）
- 記住「2 個盾牌」
- 計算正確（14 分）
- 提醒米寶回收

---

### 場景 3 — 兩個玩家輪流問 → 共佔城市各得全額分

**測試目的：** 測試多玩家共佔得分規則（規則 5）

> **設計說明：** Ollama `/api/chat` 只有 `system` / `user` / `assistant` 三種 role，無法在 API 層面區分不同玩家。解法是在訊息文字內加入 `[玩家A]` / `[玩家B]` 說話者標籤，讓模型從對話內容感知到是兩個不同的人在問問題，模擬真實桌遊店同桌客人輪流開口的情境。

| 輪次 | 中文 | English |
|------|------|---------|
| 1 | [玩家A] 我在那個城市放了一個騎士。 | [Player A] I placed a knight in that city. |
| 2 | [玩家B] 我也在同一個城市放了一個騎士欸！ | [Player B] I also placed a knight in the same city! |
| 3 | [玩家A] 城市完成了，總共 6 片板塊，我們兩個誰得分？得幾分？ | [Player A] The city just completed — 6 tiles total. Who scores? How many points do we each get? |

**正確答案：** 兩人各得 12 分（6×2=12）。米寶數量相同，所有人都得全額分數，不是平分。

**評分要點：**
- 理解兩個玩家都有騎士
- 正確說明「各得 12 分」（不是平分成各 6 分）
- 計算正確（6×2=12）
- 說明米寶數量相同時的規則

---

### 場景 4 — 修道院完成計分

**測試目的：** 測試特殊建築計分與「周圍 8 格」的定義理解

| 輪次 | 中文 | English |
|------|------|---------|
| 1 | 我放了一個修士在修道院。 | I placed a monk in a monastery. |
| 2 | 現在修道院周圍（鄰近的 8 格）已經放了 6 片板塊了。 | Now the monastery has 6 tiles placed around it (in the 8 adjacent spaces). |
| 3 | 我又補了 2 片，把修道院周圍都填滿了，這樣我可以得幾分？ | I placed 2 more tiles to fill all the surrounding spaces — how many points do I get? |

**正確答案：** 9 分（修道院本身 1 分 + 周圍 8 片 = 9 分）。修士回到手上。

**評分要點：**
- 記住「修道院」
- 理解「周圍 8 片填滿」
- 計算正確（9 分，不是 8 分）
- 提醒米寶回收

---

### 場景 5 — 農夫計分，含未完成城市情境

**測試目的：** 測試農夫只計算「已完成」城市的規則細節

| 輪次 | 中文 | English |
|------|------|---------|
| 1 | 我有個農夫在草原上，遊戲快結束了。 | I have a farmer on a field — the game is almost over. |
| 2 | 這片草原連到 2 個完成的城市，還有 1 個城市還沒完成。 | This field connects to 2 completed cities and 1 unfinished city. |
| 3 | 遊戲結束了，那個沒完成的城市最後也沒人補完，農夫得幾分？ | The game ended and that unfinished city never got completed. How many points does my farmer score? |

**正確答案：** 6 分（只算 2 個已完成城市 × 3 分，未完成城市不計入）。

**評分要點：**
- 正確說明只有已完成城市才算
- 未完成城市不計入（答案是 6 分，不是 9 分）
- 計算正確（6 分）
- 理解農夫只在遊戲結束時計分

---

### 場景 6 — 米寶放置限制

**測試目的：** 測試「只有當前玩家才能放」以及「每回合只能放一個」兩個限制

| 輪次 | 中文 | English |
|------|------|---------|
| 1 | 我剛放下一片板塊，這片板塊有城市也有草原。 | I just placed a tile that has both a city and a field on it. |
| 2 | 我朋友說他可以幫我在草原放農夫，這樣可以嗎？ | My friend says he can place a farmer on the field for me — is that allowed? |
| 3 | 那我自己可以同時在城市放騎士、在草原放農夫嗎？ | Can I place both a knight in the city AND a farmer in the field at the same time? |

**正確答案：** 兩個都不行。朋友不能在別人的回合放米寶；自己每回合也只能放一個米寶。

**評分要點：**
- 正確說明朋友不能放（只有當前玩家才能放）
- 正確說明自己只能放一個（不能同時放兩個）
- 不混淆「不同回合各自放」的情況
- 語氣自然親切

---

### 場景 7 — 遊戲結束，未完成建築計分

**測試目的：** 測試遊戲結束時未完成建築的半價計分邏輯

| 輪次 | 中文 | English |
|------|------|---------|
| 1 | 我有一個城市還沒完成，有 5 片板塊，裡面還有 2 個盾牌標記。 | I have an incomplete city with 5 tiles and 2 shield tokens inside. |
| 2 | 遊戲結束了，這個城市要怎麼算分？ | The game ended — how does this city score? |

**正確答案：** 7 分（5 片 × 1 分 + 2 盾 × 1 分）。米寶回到手上。

**評分要點：**
- 理解未完成時每片板塊與每個盾牌各 1 分（不是 2 分）
- 計算正確（7 分）
- 提醒米寶回收

---

### 場景 8 — 道路爭奪，對方無米寶

**測試目的：** 測試道路歸屬判斷，對方延伸道路但未放米寶

| 輪次 | 中文 | English |
|------|------|---------|
| 1 | 我有一個強盜在一條道路上。 | I have a robber on a road. |
| 2 | 對方接了 2 片進來，但他沒有放米寶。 | My opponent added 2 more tiles to the road but didn't place a meeple. |
| 3 | 道路完成了，總共 4 片，我朋友說他也應該得分，對嗎？ | The road completed — 4 tiles total. My friend says he should also score. Is he right? |

**正確答案：** 只有你得 4 分。對方沒有放米寶，不得分。強盜回到手上。

**評分要點：**
- 正確說明只有你得分
- 說明對方無米寶所以不得分
- 計算正確（4 分）
- 提醒強盜（米寶）回收

---

### 場景 9 — 大城市複合計算，要求列式說明

**測試目的：** 測試多步驟計算能力與字數限制下的清晰表達

| 輪次 | 中文 | English |
|------|------|---------|
| 1 | 我完成一個超大城市，8 片板塊。 | I completed a massive city — 8 tiles. |
| 2 | 裡面有 3 個盾牌。 | It has 3 shields inside. |
| 3 | 算給我聽，要列出來喔，總共幾分？ | Break down the calculation for me — how many points total? |

**正確答案：** 22 分（8×2=16 + 3×2=6）。需清楚列出計算式。米寶回到手上。

**評分要點：**
- 記住「8 片」+「3 盾牌」
- 計算正確（22 分）
- 清楚列出計算式
- 回答在字數限制內

---

### 場景 10 — 跨主題連續問，道路與城市同時追蹤

**測試目的：** 測試模型同時維護兩個建築的資訊，不能互相混淆

| 輪次 | 中文 | English |
|------|------|---------|
| 1 | 我有一條 3 片的道路快完成了，還有一個 4 片的城市也快好了，裡面有 1 個盾牌。 | I have a road almost done — 3 tiles, and also a city almost done — 4 tiles with 1 shield. |
| 2 | 道路完成了。 | The road just completed. |
| 3 | 城市也完成了，騎士在上面，這兩個加起來總共幾分？ | The city completed too, with my knight on it. What's the total score for both? |

**正確答案：** 13 分。道路：3×1=3 分。城市：4×2+1×2=10 分。兩件米寶都回到手上。

**評分要點：**
- 道路計算正確（3 分）
- 城市計算正確（10 分，含盾牌）
- 總分正確（13 分）
- 兩個建築的資訊沒有混淆

---

## 研究問題

這份測試預計回答以下幾個問題：

**Q1 — 英文版是否明顯優於中文版？**  
預期英文版在規則正確性上較高，因為英文訓練資料中關於卡卡頌的內容更豐富。若差距超過 30%，考慮在 Project Akka 加入雙語 prompt 策略。

**Q2 — gemma3:12b 本地部署是否值得？**  
前一輪測試顯示 gemma3:12b 表現優於 qwen3:14b，在更嚴謹的場景設計下驗證這個結論是否成立，並評估在 Jetson Orin Nano 上的記憶體與延遲表現。

**Q3 — 場景 3（共佔得分）在規則完整給出後，哪些模型能通過？**  
System Prompt 已補入規則 5，可以區分「規則套用能力」與「訓練資料先備知識」的差異。

**Q4 — 多步驟計算（場景 9、10）的參數量門檻在哪裡？**  
根據前一輪觀察，8b 以下模型容易出現資訊混淆，這份測試提供更明確的數據依據。

---

## 執行方式

```bash
# 全部模型、全部場景、中英文
python3 llm_test_level3_v2.py

# 指定模型
python3 llm_test_level3_v2.py --models gemma3:12b

# 只跑中文
python3 llm_test_level3_v2.py --lang zh

# 指定場景
python3 llm_test_level3_v2.py --scenarios 3,5,6

# 指定輸出檔名
python3 llm_test_level3_v2.py --output my_results.md
```

輸出報告：`akka_test_report_level3_v2_all_YYYY-MM-DD.md`

---

## 相關檔案

| 檔案 | 說明 |
|------|------|
| `llm_test_level3_v2.py` | 測試執行腳本 |
| `akka_test_report_level3_v2_*.md` | 測試輸出報告（自動產生） |
