# 煎福鍋貼專賣 · 品牌官網 PWA

極簡現代風的鍋貼店單頁式官網，含線上點餐、購物車、結帳、冷凍禮盒、門市資訊、FAQ，
並以 **Google Apps Script + Google Sheet** 作為訂單後端，附即時訂單管理看板。
純靜態、免建置，可直接部署到 GitHub Pages。

## 專案結構

```
.
├── index.html            前台官網（Tailwind CDN + Vanilla JS，單檔）
├── admin.html            訂單管理看板（需 ADMIN_KEY 登入）
├── manifest.webmanifest  PWA 設定
├── sw.js                 Service Worker（離線快取）
├── icons/                PWA 圖示（真實鍋貼照片＋發光效果）
├── gas/Code.gs           Google Apps Script 後端（貼到試算表的 Apps Script）
├── database/
│   ├── 001_create_orders.sql  Supabase 資料表建立（課堂下一步使用，網站尚未串接）
│   └── README.md              資料庫說明
├── supabase/seed.sql     Supabase 門市與商品種子資料（課堂下一步使用）
└── .nojekyll             讓 GitHub Pages 原樣提供檔案
```

## 一、部署後端（Google Sheet + Apps Script）

1. 建立新的 Google 試算表 → **擴充功能 → Apps Script**。
2. 把 `gas/Code.gs` 全部貼上並存檔。
3. 修改 `CONFIG`：
   - `ADMIN_KEY`：後台登入金鑰，請設一組長且隨機的字串（**不要提交到 GitHub**，只存在你的 Apps Script 裡）。
   - `OWNER_EMAIL`：收新訂單通知的信箱（可留空）。
4. 在編輯器選函式 `setup` → 執行並授權（建立「訂單」「訂單明細」「商品」工作表）。
5. **部署 → 新增部署作業 → 網頁應用程式**：執行身分「我」、存取權「所有人」。複製 `/exec` 網址。
6. 之後修改 Code.gs 要用「管理部署作業 → 編輯 → 新版本」重新部署，網址不變。

驗證：瀏覽器開啟 `/exec` 網址，應看到 `{"ok":true,...}`。

## 二、連接前台

編輯 `index.html`，找到下面這行，把佔位文字換成你自己的 `/exec` 網址：

```js
const GAS_URL = '請貼上你的 Apps Script 網址';
// 換成例如：const GAS_URL = 'https://script.google.com/macros/s/（你的部署代碼）/exec';
```

`GAS_URL` 還是佔位文字時為**示範模式**：送單不會真的送出（訂單編號結尾 `-DEMO`，不會寫入試算表），訂單查詢也無法使用。
此網址本身是公開端點，可放在前端；後台操作則需 ADMIN_KEY。

### 訂單查詢

前台導覽列的「訂單查詢」讓顧客輸入訂單編號,查看品項、金額、狀態與更新時間,並可按「重新查詢」取得最新狀態。
- 後端公開動作 `track` 只回傳品項/金額/狀態/時間,不含姓名、手機、地址。
- 「訂單」工作表第 14 欄為 **更新時間**(下單與後台改狀態時自動寫入);舊訂單此欄為空,顯示建立時間。
- 升級既有部署:更新 `Code.gs`、在「訂單」工作表 N1 補上標題「更新時間」(或重跑 `setup()`;它會重設「商品」工作表,自訂過價格請先備份),再「管理部署作業 → 編輯 → 新版本」重新部署。

## 三、發佈到 GitHub Pages

```bash
git init
git add .
git commit -m "init: 煎福鍋貼官網"
git branch -M main
git remote add origin https://github.com/<你的帳號>/<repo>.git
git push -u origin main
```

GitHub 專案頁 → **Settings → Pages → Build and deployment**：Source 選 *Deploy from a branch*，
Branch 選 `main` / `/ (root)` → Save。約 1 分鐘後網址為
`https://<你的帳號>.github.io/<repo>/`。

- 前台：`.../index.html`（或根網址）
- 後台：`.../admin.html`，輸入 `/exec` 網址與 `ADMIN_KEY`

### 本機預覽

Service Worker 與 PWA 安裝需要 http(s)，不能用 `file://`：

```bash
python -m http.server 8080
```

開啟 http://localhost:8080 。

## 四、下一步（課堂）：改用 Supabase

目前網站的送單、訂單查詢、管理看板**都還是使用 Google Apps Script**，尚未串接 Supabase。
`database/001_create_orders.sql` 與 `supabase/seed.sql` 是接下來課堂要在 Supabase 執行的資料庫地基，
說明請見 [database/README.md](database/README.md)。

## 注意事項

- **更新後看不到新版**：到瀏覽器開發人員工具 → Application → Service Workers → Unregister，並清除網站資料；或修改 `sw.js` 的 `CACHE` 版本號。
- **新增商品**：前台 `MENU`/`GIFTS` 與「商品」工作表的 ID、價格要一致，否則下單會被拒絕。金額以試算表為準。
- **安全**：金額由後端重算；後台走 POST + ADMIN_KEY；請勿把 ADMIN_KEY 寫進 repo。管理頁已加 `noindex`，但仍建議 repo 設為 Private，或使用自訂網域。
- **寄信額度**：免費 Gmail 每日約 100 封。

## 圖片與授權

- 菜單與主視覺圖片來自 [Unsplash](https://unsplash.com)（依 Unsplash License 使用），示範用途；正式上線請換成自家拍攝的照片。
- PWA 圖示由 Unsplash 的鍋貼（水餃）照片加工而成。
- 店名「煎福」、地址、電話、價格皆為虛構的示範資料。

## 技術

Tailwind CSS (CDN) · Lucide Icons (CDN) · Vanilla JS · Google Apps Script · Service Worker
