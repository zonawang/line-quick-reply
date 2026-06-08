# 🔮 LINE Bot 互動體驗優化實錄：打造動態智慧追問 Quick Reply 與專屬圖文選單

繼上次我們跟 AI 助理合作，為我們的 LINE 占星助理接上 Firestore，讓它擁有「永久記憶」之後，機器人基本上已經很聰明了。

在我們之前的系列旅程中，我們已經為這個 LINE Bot 奠定了堅實的基礎：
* **首部曲**：[程式小白也能看懂！我與 AI 攜手 20 分鐘無痛打造 LINE Bot實錄](https://medium.com/@zonawang/%E4%B8%8D%E6%87%82%E7%A8%8B%E5%BC%8F%E4%B9%9F%E8%83%BD%E7%9C%8B%E6%87%82-%E6%88%91%E8%88%87-ai-%E5%8A%A9%E7%90%86%E6%94%9C%E6%89%8B-20-%E5%88%86%E9%90%98%E7%84%A1%E7%97%9B%E6%89%93%E9%80%A0%E9%9B%B2%E7%AB%AF-line-%E6%A9%9F%E5%99%A8%E4%BA%BA%E5%AF%A6%E9%8C%84-a8977432d84b)
* **二部曲**：[從複誦到看圖說故事：我與 AI 助理 15 分鐘將 LINE Bot 升級 Gemini 2.5 多模態大腦與免密認證實錄](https://medium.com/@zonawang/%E7%BA%8C%E9%9B%86-%E5%BE%9E%E5%AD%B8%E8%88%8C%E5%88%B0%E7%9C%8B%E5%9C%96%E8%AA%AA%E6%95%85%E4%BA%8B-%E6%88%91%E8%88%87-ai-%E5%8A%A9%E7%90%86-15-%E5%88%86%E9%90%98%E5%B0%87-line-bot-%E5%8D%87%E7%B4%9A-gemini-2-5-%E5%A4%9A%E6%A8%A1%E6%85%8B%E5%A4%A7%E8%85%A6%E8%88%87%E5%85%8D%E5%AF%86%E9%80%9A%E9%97%9C%E5%AF%A6%E9%8C%84-9fe9c64d1ea2?postPublishedType=repub)
* **三部曲**：雖然這章還沒正式發佈，但我們成功引進了 Google ADK 框架和 Cloud Firestore，讓 Bot 不再是只有幾分鐘金魚腦的記憶。

但是在實際使用時我發現一個問題：每次機器人分析完水晶或星座，給了一大段建議後，使用者往往不知道下一步該回什麼，話題很容易就這樣乾掉。

為了解決這種「被動式一問一答」的互動僵局，我決定再次召喚我的 AI 神隊友 **Google Antigravity**，幫我的 Bot 來個互動感拉滿的升級！

這次的優化有兩個重點：
1. **動態智慧追問（Quick Reply）**：讓 Gemini 根據剛才的對話內容，自動幫使用者想 3 個最可能想問的問題，變成膠囊按鈕放在螢幕下方，點了就能直接繼續聊。
2. **專屬圖文選單（Rich Menu）**：做一個好看的選單。左邊點擊一鍵直達「使用指南」，右邊點擊則直接連到我的 [GitHub 倉庫](https://github.com/zonawang/zona-ai-learning-lab)。

雖然想法很單純，但在實作過程中，我們還是踩到了好幾個雷。下面就來記錄一下我和 **Google Antigravity** 是怎麼逐步排除萬難解決這些問題的。

---

📸 **[ 建議在此處插入優化前後的 LINE Bot 互動對比圖，例如：Quick Reply 膠囊按鈕與 Rich Menu 實際手機畫面 ]**

---

## 🧩 第一關：Gemini 動態追問與 LINE 20 字的嚴格限制

在對話式 UI 中，使用者常面臨「接下來該問什麼」的瓶頸。為了主導引導對話，我們利用 Gemini 的推理能力，在產生主要回覆後，即時分析上下文並生成三個最具相關性的口語化追問。

這個想法用 Gemini 來做其實很簡單，但在串接 LINE API 時卻立刻踢到了鐵板：

> **LINE API 規定：Quick Reply 按鈕的標籤（Label）字數限制，最大只能容納 20 個字！**

只要 Gemini 產生的問題多出一個字，整批 Quick Reply 就會直接發送失敗，手機畫面上什麼都看不到。

### 💡 解決方案：Prompt 限制與程式碼雙重截斷

這時候我和 **Google Antigravity** 討論後，決定用雙重防禦機制來解這個字數限制：

#### 1. 在 Prompt 中先發制人
直接在發送給 Gemini 的指令（Prompt）中加入字數硬性限制：
```text
每個追問問題必須非常簡短、口語，且嚴格限制在 20 個字以內。
```

#### 2. 在 JavaScript 程式碼中強制防禦
為了防止模型偶爾超出字數，我們在程式碼中加上了強制的安全截斷與整理邏輯：
```javascript
const quickReplies = questions.map(q => {
  // 強制限制在 20 字內，並去除多餘空格
  const cleanLabel = q.trim().substring(0, 20);
  return {
    type: 'action',
    action: {
      type: 'message',
      label: cleanLabel,
      text: cleanLabel
    }
  };
});
```

調整完之後，追問按鈕就能穩定的在手機下方顯示了，使用者輕輕一點就可以順著話題直接問下去。

---

## 🧩 第二關：手繪選單與突破 LINE 1MB 上限挑戰

接下來是圖文選單。我一開始對 **Google Antigravity** 說：

> 💬 *「我要使用 richmenu.png 作為我的richmmenu,左邊是這個line bot 的使用指南，右邊請連到https://github.com/zonawang/zona-ai-learning-lab」*

Antigravity 很快就幫我寫好了註冊 Rich Menu 的腳本。但後來我發現圖片給錯了，連忙跟它說：

> 💬 *「圖片不對欸，請把圖片改成 123.png」*

這張 `123.png` 是我設計的手繪神秘卡牌風格的圖，效果非常好。

📸 **[ 建議在此處插入手繪卡牌風格的圖文選單設計圖 123.png ]**

但問題來了，這張圖因為解析度高，大小有 **8.1 MB**！
即便我們用工具把它縮放到 LINE 規定的 Large 尺寸 (2500x1686)，因為 PNG 是無損壓縮，檔案大小還是高達 7.5 MB。

當我們嘗試上傳到 LINE 時，伺服器直接回傳了 `413 Request Entity Too Large` 的報錯：

> **LINE 的 Rich Menu 圖片檔案上限是 1MB。**

### 💡 解決方案：sips 轉檔壓縮與自動化註冊腳本

對此，Antigravity 給出了解方。我們直接利用 macOS 內建的 `sips`（Scriptable Image Processing System）工具進行圖片格式轉換與品質壓縮：

```bash
sips -s format jpeg -s formatOptions 70 -z 1686 2500 123.png --out richmenu_resized.jpg
```

這個指令在幾乎看不出畫質損耗的情況下，把 8.1MB 的巨圖壓到了 **955 KB**，順利通過 1MB 的門檻！

接著，我們用 `create-rich-menu.js` 腳本，透過 Node 22 原生的 `fetch` API 完成了「一鍵註冊選單 -> 上傳 JPEG 圖片 -> 設定為全球預設」的完整流程，直接把我的手繪選單放上了 LINE：

```javascript
// create-rich-menu.js 核心邏輯片段
const response = await fetch('https://api.line.me/v2/bot/richmenu', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${CHANNEL_ACCESS_TOKEN}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(richMenuDefinition)
});
```

---

## 🧩 第三關：點了選單沒反應？抓出 JavaScript 的隱藏 Bug

選單上線後，左邊的「使用指南」應該要能直接秒出說明書。因為這部分是固定的文字，為了省 Token 和提升回應速度，我不想讓它跑 LLM 運算，而是直接在後端攔截文字並回傳。

結果，上線測試時卻發現：

> 💬 *「點了richmenu左側的使用指南，沒反應」*

怎麼會這樣？

我們翻了一下 Cloud Run 的日誌，發現伺服器拋出了 `ReferenceError: isGuide is not defined` 的錯誤。
魔鬼藏在程式碼的變數作用域裡：

```javascript
try {
  let isGuide = false; // ❌ 宣告在 try 區塊內！
  if (userMessage === '使用指南') {
    isGuide = true;
    responseText = "🔮 歡迎使用水晶占星助理！...";
  }
} catch (err) {
  console.error(err);
}

// ❌ 報錯！isGuide 在 try 區塊外部無法被存取
if (isGuide) { 
  await replyMessage(replyToken, responseText);
}
```

在 JavaScript 中，使用 `let` 宣告的變數具有**區塊作用域（Block Scope）**。變數 `isGuide` 被宣告在 `try` 區塊內部，當我們在 `try-catch` 結構外部引用該變數時，便會觸發 `ReferenceError`，導致 Webhook 在發送回覆前異常中斷。

### 💡 解決方案：變數提升至函式層級

知道原因後就好解了。我們迅速重構，把 `isGuide` 的宣告移至 `handleEvent` 函式的最上層，修正了作用域問題：

```javascript
let responseText = '';
let isGuide = false; // ✅ 正確宣告在函式層級！

try {
  if (userMessage === '使用指南') {
    isGuide = true;
    responseText = "🔮 歡迎使用水晶占星助理！...";
  }
} catch (err) {
  console.error(err);
}

if (isGuide) {
  await replyToLine(replyToken, responseText); // 0 毫秒極速直出！
}
```

重新部署後，點擊「使用指南」就能在 0 毫秒、0 Token 的情況下秒出說明書了！

---

## 🧩 第四關：Push 程式碼與完成部署

一切測試正常後，我對 Antigravity 說：

> 💬 *「請先把以上push上去這個repo」*

它也非常配合，迅速幫我整理好修改過的 `index.js`、`create-rich-menu.js` 等所有檔案，連同壓縮後的 `richmenu_resized.jpg` 圖片，一口氣 commit 並安全地 Push 到了我們的 GitHub Repo 上。

---

## 💬 結語：人機協作推動高效開發

這次跟 **Google Antigravity** 協同開發的體驗依然很棒。

* **AI 負責解決各種底層技術與問題診斷**：比如提供 `sips` 壓縮參數、處理 Node 22 原生的 Fetch 呼叫，還有幫我快速抓出那個隱藏的 `let` 變數區塊作用域 Bug。
* **我則負責掌握大方向與用戶體驗**：調整版面、給出具體需求、並在遇到問題時即時提供給 AI 具體的回饋。

用這種方式開發專案，我們不再需要花費大量時間去死記各種複雜的指令，而是能將精力集中在「如何清晰表達邏輯與創意」，讓想法在最短時間內精準落地。

---

本專案的完整程式碼、安全配置、1MB 影像壓制方案與 Gemini Quick Reply 實作已全數備份至最新 GitHub 倉庫：https://github.com/zonawang/line-quick-reply.git

如果您對建立自己的智慧型 LINE 機器人或與 AI 協同開發感興趣，歡迎隨時與我交流！
