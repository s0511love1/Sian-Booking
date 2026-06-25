# 單頁 HTML 應用開發標準書
**適用範圍：個人工作室 / 小型商業後台管理系統**
**技術棧：純 HTML + CSS + JavaScript（無框架）+ Google Sheets API**
**版本：1.0 ｜ 2026-06-26**

---

## 一、架構設計原則

### 1-1 系統角色分工

| 層級 | 工具 | 職責 |
|---|---|---|
| 前端 | 單一 HTML 檔案 | UI 渲染、使用者互動、資料讀寫邏輯 |
| 部署 | GitHub + Vercel | 讓 HTML 有公開網址，可在手機開啟 |
| API 層 | Google Apps Script | 接收前端請求、讀寫試算表、驗證密鑰 |
| 資料庫 | Google Sheets | 儲存所有業務資料 |
| 快取 | LocalStorage | 離線備用、加速初次渲染 |

### 1-2 資料流架構

```
手機開啟 App
    ↓
1. 先從 LocalStorage 快速渲染（零延遲，避免白畫面）
    ↓
2. 呼叫 Apps Script API 同步最新資料（GET 請求）
    ↓
3. 更新畫面 + 寫回 LocalStorage 快取
    ↓
使用者操作（新增/修改/刪除）
    ↓
4. 即時更新畫面 + LocalStorage
    ↓
5. 非同步寫入 Google Sheets（失敗只警告不阻擋）
```

### 1-3 單一 HTML 檔案規範

- 所有 CSS、JS 寫在同一個 `index.html`（方便 GitHub 單檔部署）
- 外部資源只引用 CDN（字體、圖示庫），且全部採**延遲載入**（點擊時才載入，非頁面載入時）
- 檔名統一為 `index.html`，直接上傳 GitHub 即可覆蓋部署

---

## 二、Google Apps Script API 規範

### 2-1 跨域呼叫標準（CORS）

**核心問題**：Apps Script Web App 不支援瀏覽器 `fetch()` 直接 POST，原因是 Google 伺服器在 POST 時會進行 302 重新導向，觸發 CORS preflight，導致 `Failed to fetch` 錯誤。

**標準解法：使用 GET + 參數傳遞 JSON Payload**

```javascript
// ✅ 正確寫法（CORS 安全）
async function callAPI(payload) {
  const fullPayload = { ...payload, key: SECRET_KEY };
  const url = API_URL + '?p=' + encodeURIComponent(JSON.stringify(fullPayload));
  const res = await fetch(url, { method: 'GET', redirect: 'follow' });
  const data = await res.json();
  if (!data.ok) throw new Error(data.error || 'API 錯誤');
  return data;
}

// ❌ 錯誤寫法（會觸發 CORS preflight，在瀏覽器環境失敗）
await fetch(API_URL, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(payload)
});
```

### 2-2 Apps Script 必須同時實作 doGet 和 doPost

```javascript
// Apps Script 端必須同時處理 GET（瀏覽器呼叫用）和 POST（備用）
function doGet(e) {
  try {
    const p = e.parameter && e.parameter.p ? JSON.parse(e.parameter.p) : {};
    return handleRequest(p);
  } catch(err) {
    return jsonResponse({ ok: false, error: err.message });
  }
}

function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    return handleRequest(data);
  } catch(err) {
    return jsonResponse({ ok: false, error: err.message });
  }
}
```

### 2-3 API 回應格式統一

```javascript
// 成功
{ ok: true, data: {...} }

// 失敗
{ ok: false, error: '錯誤說明' }

// Apps Script 回傳方式（固定格式）
function jsonResponse(obj) {
  return ContentService
    .createTextOutput(JSON.stringify(obj))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### 2-4 安全性：密鑰驗證

```javascript
// Apps Script 端
const SECRET_KEY = '你的密鑰';  // ← 每個系統改不同的

function handleRequest(data) {
  // 公開 API（不需密鑰，例如客戶填單提交）
  if (data.action === 'submitBooking') return handleSubmitBooking(data);
  if (data.action === 'ping') return jsonResponse({ ok: true });

  // 需要密鑰的 API
  if (data.key !== SECRET_KEY) {
    return jsonResponse({ ok: false, error: 'unauthorized' });
  }
  // ... 其他操作
}
```

### 2-5 部署版本管理

**⚠️ 重要：每次修改 Apps Script 程式碼後，必須重新部署新版本，否則線上跑的還是舊程式碼。**

部署步驟：
1. Apps Script → 右上角「部署」→「管理部署作業」
2. 點鉛筆圖示「編輯」
3. 版本選「建立新版本」
4. 點「部署」

---

## 三、資料持久化規範

### 3-1 雙層儲存策略

```javascript
// 所有資料操作都要同時處理兩層
async function cloudSave(action, payload) {
  // 第一層：立即寫入 LocalStorage（使用者不等待）
  saveToLocal();
  // 第二層：非同步寫入 Google Sheets（失敗不影響使用）
  if (isCloudReady()) {
    try {
      await callAPI({ action, ...payload });
    } catch(err) {
      showToast('⚠️ 雲端同步失敗，已保存在本機');
    }
  }
}
```

### 3-2 LocalStorage 規範

```javascript
const STORAGE_KEY = '系統名稱_data_v1';  // 版本號方便日後資料格式升級

function saveToLocal() {
  localStorage.setItem(STORAGE_KEY, JSON.stringify({
    orders, clients, expenses, tags,
    nextOrderId, nextExpId, nextTagId,
    savedAt: new Date().toISOString()
  }));
}

function loadFromLocal() {
  const raw = localStorage.getItem(STORAGE_KEY);
  if (!raw) return false;
  const data = JSON.parse(raw);
  // 每個欄位都要個別判斷，避免格式不符時整體崩潰
  if (data.orders)   orders   = data.orders;
  if (data.clients)  clients  = data.clients;
  // ...
  return true;
}
```

### 3-3 變數宣告規範

**所有資料陣列必須使用 `let`，禁止使用 `const`**（匯入備份功能需要整體替換陣列）

```javascript
// ✅ 正確
let orders = [];
let clients = [];
let expenses = [];
let tags = [];

// ❌ 錯誤（匯入時會報 Assignment to constant variable）
const orders = [];
const clients = [];
```

---

## 四、App 啟動流程規範

### 4-1 標準啟動序列

```javascript
async function initApp() {
  // Step 1：從本機快取快速渲染（避免白畫面）
  loadFromLocal();
  renderAll();       // 渲染所有頁面
  dismissSplash();   // 關閉啟動畫面

  // Step 2：從雲端同步最新資料
  await syncFromCloud();

  // Step 3：用雲端資料重新渲染
  renderAll();
}
initApp();
```

### 4-2 啟動畫面（Splash Screen）

iOS standalone App 模式在載入 HTML 前會有黑畫面，必須用 JS 啟動畫面遮蓋：

```html
<div id="splashScreen">
  <!-- App 名稱 + 進度條 -->
</div>
<script>
function dismissSplash() {
  const el = document.getElementById('splashScreen');
  // 先跑進度條動畫，再淡出
  setTimeout(() => { el.style.opacity = '0'; }, 700);
  setTimeout(() => { el.style.display = 'none'; }, 1100);
}
</script>
```

---

## 五、iOS PWA（加入主畫面）規範

### 5-1 必要 Meta 標籤

```html
<!-- 視窗設定：viewport-fit=cover 讓內容延伸到安全區域 -->
<meta name="viewport" content="width=device-width, initial-scale=1.0,
      maximum-scale=1.0, user-scalable=no, viewport-fit=cover">

<!-- iOS 獨立 App 模式（無 Safari 介面） -->
<meta name="apple-mobile-web-app-capable" content="yes">
<!-- ⚠️ 使用 default，不要用 black-translucent（某些機型會黑畫面） -->
<meta name="apple-mobile-web-app-status-bar-style" content="default">
<meta name="apple-mobile-web-app-title" content="App 名稱">
<meta name="mobile-web-app-capable" content="yes">
<meta name="theme-color" content="#F7F2EF">

<!-- 主畫面圖示（SVG inline data URI，不需要外部檔案） -->
<link rel="apple-touch-icon" href="data:image/svg+xml,...">
```

### 5-2 安全區域（Safe Area）CSS 規範

不同 iPhone 型號的 Home Indicator 區域大小不同，必須用 CSS 環境變數自動適配：

```css
:root {
  --nav-h: 54px;
  /* iOS 安全區域：自動偵測，不需要 media query */
  --safe-bottom: env(safe-area-inset-bottom, 0px);
  --safe-top: env(safe-area-inset-top, 0px);
}

/* 底部導覽列：高度加入安全區域 */
.bottom-nav {
  height: calc(var(--nav-h) + var(--safe-bottom));
  padding-bottom: var(--safe-bottom);
}

/* 頁面捲動區：底部留白加入安全區域 */
.page-scroll {
  padding-bottom: calc(var(--nav-h) + var(--safe-bottom) + 16px);
}

/* 固定在底部的 CTA 按鈕列 */
.cta-bar {
  padding-bottom: calc(24px + var(--safe-bottom));
}
```

### 5-3 更新主畫面圖示的完整步驟

加了新的 meta 標籤後，舊的主畫面圖示**不會自動更新**，必須：
1. 長按舊圖示 → 移除 App
2. 設定 → Safari → 進階 → 網站資料 → 刪除該網址的快取
3. 重新用 Safari 開啟網址 → 加入主畫面

---

## 六、效能規範

### 6-1 外部函式庫延遲載入

不常用的大型函式庫（如 SheetJS）不應在頁面載入時引入，改為「點擊時才載入」：

```javascript
// ✅ 正確：延遲載入
let xlsxLoadPromise = null;
function loadXLSX() {
  if (typeof XLSX !== 'undefined') return Promise.resolve();
  if (xlsxLoadPromise) return xlsxLoadPromise;
  xlsxLoadPromise = new Promise((resolve, reject) => {
    const script = document.createElement('script');
    script.src = 'https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js';
    script.onload = resolve;
    script.onerror = () => reject(new Error('載入失敗'));
    document.head.appendChild(script);
  });
  return xlsxLoadPromise;
}

// 使用時
document.getElementById('exportBtn').onclick = () => {
  loadXLSX().then(() => buildExcelReport());
};

// ❌ 錯誤：在 <head> 直接引入（每次開 App 都要下載 600KB+）
<script src="https://cdn.jsdelivr.net/npm/xlsx@0.18.5/dist/xlsx.full.min.js"></script>
```

### 6-2 避免效能洩漏的常見問題

| 問題 | 症狀 | 解法 |
|---|---|---|
| `setInterval` 未清除 | 越用越卡、電池消耗快 | 用 `clearInterval` 或改用 `setTimeout` |
| Scroll 事件重複綁定 | 滾動越來越慢 | 用旗標 `let attached = false` 防止重複綁定 |
| 表單重繪搶走焦點 | 輸入框打不出字 | 輸入值用 `onblur` 儲存，不用 `oninput` 觸發重繪 |
| 大量 DOM 節點不清除 | 列表越捲越慢 | 重繪前先 `innerHTML = ''` 清空 |

---

## 七、程式碼品質規範

### 7-1 語法檢查（每次修改後必做）

```bash
node -e "
const fs = require('fs');
const content = fs.readFileSync('index.html', 'utf8');
const js = content.slice(content.indexOf('<script>')+8, content.lastIndexOf('</script>'));
try { new Function(js); console.log('✅ JS syntax OK'); }
catch(e) { console.log('❌ ERROR:', e.message); }
"
```

### 7-2 常見 Bug 速查表

| 錯誤訊息 | 原因 | 解法 |
|---|---|---|
| `Assignment to constant variable` | 資料陣列宣告為 `const`，匯入時無法重新賦值 | 改為 `let` |
| `Identifier 'xxx' has already been declared` | 變數重複宣告（通常是大檔案分段寫入時截斷） | 用 `str_replace` 修改，寫完跑語法檢查 |
| `Failed to fetch` | Apps Script POST 請求觸發 CORS preflight | 改用 GET + 參數，`redirect: 'follow'` |
| 輸入框打不出字 | `oninput` 觸發整個表單重繪，每打一字就重繪 | 改用 `onblur`，送出時強制 `blur()` 再讀值 |
| 黑畫面（iOS PWA） | `black-translucent` 狀態列 + `viewport-fit=cover` 在某機型衝突 | 改用 `default` 狀態列設定 |
| 底部內容被遮住（不同機型） | 未處理 iOS 安全區域 | 全面使用 `env(safe-area-inset-bottom)` |

### 7-3 大檔案修改規範

- **禁止用整檔覆寫**修改超過 64KB 的 HTML（bash `cat > file << 'EOF'` 有大小限制）
- 使用 `str_replace` 精確替換目標字串
- 每次大改動後必跑語法檢查
- 改完立即測試，不要累積多個未測試的修改

---

## 八、資料模型規範

### 8-1 通用欄位設計

```javascript
// 每筆資料都應包含
{
  id: Number,           // 自增數字 ID
  createdAt: String,    // ISO 8601 時間戳
  updatedAt: String,    // 最後更新時間
}

// 有關聯關係的資料用 ID 關聯，不嵌套資料
// ✅ 正確
{ orderId: 3, amount: 5000 }

// ❌ 避免（嵌套資料難以同步更新）
{ order: { id: 3, name: '...', service: '...' }, amount: 5000 }
```

### 8-2 Google Sheets 欄位規範

- 陣列值存為「以頓號分隔的字串」（例如：`敏感肌、油性`）
- Boolean 值存為字串 `'true'` / `'false'`（Sheets 讀出來都是字串）
- 讀取時必須做型別轉換：

```javascript
function parseRow(r) {
  return {
    id:      parseInt(r.id),
    tags:    r.tags ? r.tags.split('、').filter(Boolean) : [],
    repeat:  r.repeat === 'true' || r.repeat === true,
    amt:     parseFloat(r.amt) || 0,
  };
}
```

---

## 九、版本管理規範

### 9-1 版本命名

```
系統名稱Ver.主版本號.次版本號
例：Sian後台管理Ver.01、團購系統Ver.01
```

- **主版本號**：重大架構調整（例如從 LocalStorage 改為 Google Sheets）
- **次版本號**：功能新增或 Bug 修正

### 9-2 版本標記

在 HTML 檔案最開頭加入版本說明：

```html
<!-- ============================================================
     系統名稱
     版本：Ver.XX
     建立日期：YYYY-MM-DD
     說明：本版本包含的主要功能
     ============================================================ -->
```

### 9-3 部署流程

```
修改 index.html（本機或 Claude）
    ↓
JavaScript 語法檢查（node -e "new Function(js)"）
    ↓
上傳到 GitHub（Upload file 覆蓋 index.html）
    ↓
Vercel 自動重新部署（約 30 秒）
    ↓
手機重新整理網址測試
```

---

## 十、安全性規範

### 10-1 密鑰管理

- 每個系統使用**獨立密鑰**，不同系統不共用
- 密鑰儲存在前端 `CLOUD_CONFIG` 中（不可避免，因為是靜態網站）
- 風險評估：密鑰隱藏在 JS 原始碼中，一般使用者不會去看；最大風險是有人拿到密鑰後亂寫資料，但因為有「待審核」機制，實際損害有限
- 密鑰長度建議 **16 字元以上**，包含中英文數字混合

### 10-2 個資保護

- 後台讀取端點（含客戶姓名、電話等個資）**必須要求密鑰驗證**
- 客戶填單提交端點可公開（只寫入，不讀取已有資料）
- 定期用「匯出備份」在本機保存一份資料副本

