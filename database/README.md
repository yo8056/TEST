# 資料庫(Supabase PostgreSQL)

這個資料夾是把訂單後端從「Google Apps Script + Google Sheet」搬到 Supabase 的**資料庫地基**。
目前網站(`index.html`、`admin.html`)**仍然使用 Google Apps Script**,還沒有串接 Supabase。

## 架構與安全原則

完成串接後的目標架構:

```
訪客的瀏覽器 ──► Supabase Edge Functions(使用 service_role)──► 資料庫
        └──(僅讀取)anon key ──► products / stores(只看得到上架資料)
```

- 訪客**不能**直接讀寫訂單相關資料表。訂單、明細、狀態紀錄、計數器、店員名單都啟用 RLS 且**沒有任何政策**,
  並且明確收回 anon / authenticated 的所有權限。
- 訂單資料之後要由 Edge Function 用 **service_role** 讀寫;service_role 也只被授與必要權限(**沒有 DELETE / TRUNCATE**)。
- 下單、查單、改狀態是三個資料庫函式,**只有 service_role 能執行**。
- ⚠️ service_role / secret key **只能放在 Edge Function 的環境變數**,絕對不要寫進 `index.html`、`admin.html` 或任何前端檔案。
  前端只能用 anon(publishable)key,而且它只能讀商品與門市。

## 檔案與執行順序

| 順序 | 檔案 | 用途 |
|---|---|---|
| 1 | `database/001_create_orders.sql` | 建立資料表、限制、索引、函式、觸發器,並設定 RLS 與權限 |
| 2 | `supabase/seed.sql` | 匯入門市(3 間)與商品(19 項)的初始資料 |
| — | `database/README.md` | 本說明 |

在 Supabase Dashboard → **SQL Editor** 貼上整份檔案後執行(請用預設的 postgres 身分)。**先 001,再 seed。**

> 001 可以重複執行(只用 `create ... if not exists` / `create or replace`,觸發器與政策是先 drop 再重建),
> 不使用 DROP TABLE、TRUNCATE,也不會刪除任何資料列。
>
> seed.sql 是一般的 `insert`,**只要執行一次**。重複執行會出現 `duplicate key` 錯誤,代表資料已經匯入過,不會產生重複資料。

## 001 建立了什麼

**7 張資料表**

| 資料表 | 用途 |
|---|---|
| `stores` | 門市(代碼、名稱、地址、電話、是否營業、排序) |
| `products` | 商品(代碼、品名、價格、分類、描述、圖片、標籤、是否上架、排序) |
| `orders` | 訂單(`order_no`、狀態、取餐方式、顧客資料、門市、金額) |
| `order_items` | 訂單明細(品名與單價為下單當下的快照) |
| `order_status_log` | 狀態變更紀錄(誰、何時、從哪到哪) |
| `order_counters` | 每日訂單流水號計數器 |
| `staff_members` | 店員名單(之後的後台用來判斷登入者是否為店員) |

**資料庫函式**

| 函式 | 用途 |
|---|---|
| `create_order(payload jsonb)` | 下單:驗證欄位、以 `products` 價格重算金額、產生訂單編號,回傳 `order_no`、`total`、`eta` |
| `track_order(p_order_no text)` | 查單:只憑訂單編號,只回傳品項、金額、狀態、時間,不回傳個資 |
| `update_order_status(p_order_no, p_status, p_actor)` | 後台改狀態(會寫入狀態紀錄) |
| `calc_fee`、`next_order_no`、`is_staff` | 內部輔助函式(運費/服務費、訂單編號、店員判斷) |

**代碼**

- 訂單狀態:`new`(新訂單)、`preparing`(製作中)、`ready`(待取餐)、`delivering`(配送中)、`done`(已完成)、`cancelled`(已取消)
- 取餐方式:`dine`(內用)、`take`(外帶)、`ship`(宅配)
- 訂單編號:`JF` + `yyMMdd`(台北時間)+ 流水號,例如 `JF251022-001`

## seed.sql 匯入了什麼

- `stores`:`flag`(旗艦店)、`xinyi`(信義店)、`taichung`(台中勤美店)
- `products`:`s1`–`s4`(招牌鍋貼)、`d1`–`d4`(精選水餃)、`p1`–`p4`(湯品/飲料)、`k1`–`k4`(經典小菜)、`g1`–`g3`(冷凍禮盒)

這些代碼與價格和 `index.html` 的 `STORES` / `MENU` / `GIFTS`、`gas/Code.gs` 的 `PRODUCTS` 一致。

## 每一步預期看到什麼

**執行 001**:最後顯示 *Success. No rows returned*,沒有錯誤。
之後在 Table Editor 應看到 7 張表:`stores`、`products`、`orders`、`order_items`、`order_status_log`、`order_counters`、`staff_members`,
且每張表都標示 RLS enabled。

**執行 seed.sql**:顯示 *Success*。之後在 Table Editor 應看到 `stores` 有 3 筆、`products` 有 19 筆;
其餘 5 張訂單相關的表是空的。

## 哪些來自原專案,哪些是新設計

### 來自原專案(`gas/Code.gs`、`index.html`)

| 項目 | 來源 |
|---|---|
| 訂單欄位:訂單編號、建立時間、狀態、取餐方式、姓名、手機、Email、門市、地址、備註、小計、運費/服務費、應付總額、更新時間 | 「訂單」工作表 14 欄 |
| 明細欄位:商品ID、品名、單價、數量、小計(品名與單價為下單當下快照) | 「訂單明細」工作表 |
| 商品欄位:ID、品名、單價、分類、上架 | 「商品」工作表 |
| 驗證規則:手機 `09` 開頭共 10 碼、Email 格式、姓名必填、宅配必填地址、1–30 個品項、每項 1–50 份、文字截 200 字、清除控制字元 | `validate_`、`price_` |
| 費用規則:內用 10%(四捨五入)、外帶 0、宅配滿 1000 免運否則 150 | `FEES` |
| 金額以資料庫商品價格重算,不信任前端 | `price_` |
| 訂單編號格式 `JF` + `yyMMdd` + 流水號 | `nextOrderNo_` |
| honeypot 欄位:機器人靜默略過 | `doPost` |
| 查單只憑訂單編號,不回傳個資 | `track_` |
| 狀態共六種(新訂單、製作中、待取餐、配送中、已完成、已取消) | `CONFIG.STATUSES` |
| 預計時間文字(15–20 分鐘 / 2–3 個工作天) | `FEES[..].eta` |

### 新設計或改變

| 項目 | 說明 |
|---|---|
| `id uuid` 主鍵 + 唯一 `order_no` | 原本訂單編號就是識別,現在 uuid 為主鍵、`order_no` 為顯示用且唯一 |
| 狀態改英文代碼 | `new / preparing / ready / delivering / done / cancelled`,畫面再轉成中文 |
| 取餐方式改代碼 | `dine / take / ship`(前端本來就送這三個值) |
| `stores` 表、`store_id` | 原本訂單存「旗艦店 — 地址」整串文字,現在存門市代碼 |
| `products` 表加入圖片、描述、標籤、排序等 | 原本只有價格在 Sheet,其餘寫死在前端 |
| `order_status_log` 狀態紀錄 | 原本只有最後狀態;現在記錄每次變更(誰、何時、從哪到哪) |
| `order_counters` 每日計數器 | 原本存在 Apps Script 的 Script Properties;並修正「第 1000 筆會重複」的問題 |
| `staff_members` 店員名單 | 取代 `ADMIN_KEY`;之後的後台以此判斷是否為店員 |
| `updated_at` 自動更新(觸發器) | 原本靠程式手動寫入 |
| 時間改為 `timestamptz`,編號日期固定用台北時間 | 原本 Sheet 時間不含時區 |
| 同一商品重複出現時合併成一列 | 原本會變成兩列 |
| 內用 / 外帶必須選門市 | 原本後端沒有強制;資料表本身仍允許 `store_id` 為空,以便遷移舊資料 |
| 資料庫層的 CHECK 限制(`total = subtotal + fee` 等)、RLS 與函式權限 | 全新 |
