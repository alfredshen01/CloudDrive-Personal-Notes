# 前置期日誌（2026-06）：Final_Project 改版

彩色轉黑白圖片轉換網站(由原「日曆」專案改版而來),規劃為可擴充的多功能圖片處理平台。

## 專案目標

| 目標 | 狀態 |
|------|------|
| 使用者註冊／登入／登出 | ✅ |
| 上傳彩色照片轉成黑白並下載 | ✅ |
| 只接受圖片、限制檔案大小 | ✅ |
| 直式照片自動轉正 | ✅ |
| 下載可選無損(PNG)／有損(JPG)畫質 | ✅ |
| 轉換後檔案保存,可隨時回來下載 | ✅ |
| 側邊資料夾分類(支援日後新功能) | ✅ |
| 新增其他圖片處理功能(縮圖、濾鏡…) | 🔲 |

---

## 2026-06-02

### 完成項目
- 將專案從「日曆」改版為「彩色轉黑白」圖片轉換網站,保留登入功能
- 改版前先打 git tag `v0.1-before-overhaul` 保存舊狀態,確認重要檔案都有進 git
- 後端用 Pillow 做灰階轉換(`/convert`),移除舊的 events API
- 修正中文檔名上傳會 500 的問題
- nginx 加 `client_max_body_size 12M`,避免大圖被擋
- 介面改為極簡高級灰階風,品牌改名 **Final_Project**
- 新增「檔案儲存 + 分類系統」:`files` 資料表、Docker volume 存檔、列表／下載／刪除 API、側邊資料夾 UI
- 修正直式照片轉出來變橫的(EXIF 方向)
- 效能優化:預覽改用有損 JPEG(快),下載可選無損 PNG／有損 JPG
- 上傳介面精簡成矮橫條
- 清理沒在用的死檔 `initdb/`

### 遇到的問題

**問題 1:沿用舊模板改版,擔心資料庫會變亂**
- 緣由:直接拿「日曆」專案的模板改成圖片網站,怕有殘留的東西沒清乾淨。
- 原因:舊專案的 schema 同時被兩個地方描述——`initdb/01_create_tables.sql`(原始 SQL)和 alembic migration。其中 `initdb/` 根本沒被 docker-compose 掛載,是**死檔**,卻會讓人誤以為它有效;另外 `events` 表已不再使用但仍留在 DB。同一份資料有兩個來源,正是「會亂」的根源。
- 解決方法:確認 alembic 才是唯一真實來源後刪除 `initdb/`;`events` 閒置表暫時保留(日後要清就新增一支 `DROP TABLE` migration,而非回頭改舊檔)。

**問題 2:上傳顯示「連線失敗」,實際是中文檔名造成 500**
- 緣由:上傳圖片時前端顯示「連線失敗,請稍後再試」,換瀏覽器、用無痕都一樣。
- 原因:這題繞了三層才找到真因,過程本身是重點——
  1. 一開始以為是**部署空檔的 502**(容器重建時 API 短暫下線),但部署完仍失敗;
  2. 接著懷疑 **CORS / www 跨源**,實測同源請求其實正常,排除;
  3. 用瀏覽器 DevTools 看到真正是 **HTTP 500**。最終定位:使用者照片檔名含**中文**,後端把它放進 `Content-Disposition` HTTP header,而 HTTP header 只能用 latin-1 編碼,中文無法編碼 → 拋出 `UnicodeEncodeError` → 未捕捉的 500。前端的 catch 把「回應不是 JSON」也一律報成「連線失敗」,才會誤導方向。
- 解決方法:`Content-Disposition` 改用 RFC 5987 寫法(ASCII 安全的 `filename` + UTF-8 的 `filename*`);前端錯誤訊息改成會帶 HTTP 狀態碼,避免再把不同錯誤混為一談。

**問題 3:直式照片轉出來變橫的**
- 緣由:手機拍的直式照片,轉黑白後變成橫躺的。
- 原因:手機感光元件其實是橫的,直式照片是靠 EXIF 的 Orientation 標記告訴看圖軟體「顯示時要轉 90 度」。Pillow 預設**不會自動套用**這個標記,而且轉成 PNG 後 EXIF 也被丟掉,所以顯示的是未旋轉的原始橫向像素。
- 解決方法:轉換前先用 `ImageOps.exif_transpose()` 依 EXIF 把像素實體轉正,再轉灰階。

**問題 4:以為轉換慢是伺服器 CPU 不好**
- 緣由:轉一張圖要等十幾秒,直覺以為是 Droplet 處理器太弱。
- 原因:實測純運算(解碼+轉灰階+編碼)只要約 0.1~1 秒,根本不是瓶頸。真正的時間花在**資料傳輸**:一次要上傳約 8MB 原圖 + 下載結果。而結果之所以大,是因為輸出的是**無損 PNG**,一張照片的灰階 PNG 往往跟原圖一樣大(6.5MB)。升級 CPU 幫助不大,該做的是縮小傳輸量。
- 解決方法:預覽改回傳有損 JPEG(小很多、顯示快);下載讓使用者自己選無損 PNG 或有損 JPG。

**問題 5:刪除測試帳號時被外鍵約束擋住**
- 緣由:`DELETE FROM users WHERE id IN (...)` 報錯 `violates foreign key constraint ... on table events`。
- 原因:`events` 表的 `user_id` 還指向那些使用者,資料庫不允許刪掉「仍被引用」的列,否則會留下孤兒資料。這也是舊模板殘留的 `events` 表造成的副作用。
- 解決方法:先刪子資料(`DELETE FROM events WHERE user_id IN (...)`)再刪使用者;新建的 `files` 表則直接設 `ondelete=CASCADE`,以後刪使用者會連帶清掉其檔案記錄。

**問題 6:上傳區被撐到幾乎滿版**
- 緣由:把上傳區改小後,實際畫面上它還是佔滿整個視窗,結果圖被擠到最下面。
- 原因:第一版只改了內距(padding),但 antd 的 Dragger 元件 `.ant-upload-drag` 預設帶 `height: 100%`,被它撐高;而且調整 flex 時下錯層級(真正包內容的是 `.ant-upload-drag-container` 不是 `.ant-upload-btn`),所以圖示和文字也沒排成橫排。
- 解決方法:對 `.ant-upload.ant-upload-drag` 設 `height: auto !important` 覆蓋預設,並把 flex 規則下到 `.ant-upload-drag-container`,做成矮橫條。

### 待釐清
- `events` 閒置表要不要正式用 migration 移除(目前留著無害)。
- 檔案目前存在 Droplet volume,若日後量大或要防伺服器掛掉,再評估改用 DigitalOcean Spaces(S3)。

### 待辦
- [ ] 清掉除錯時建立的測試帳號(`_tmptest_*`、`_verify_*`、`_filetest_*`、`_exiftest_*`、`_uitest_*`):`DELETE FROM users WHERE username LIKE '\_%';`
- [ ] 規劃第二個圖片處理功能(例如縮圖或濾鏡),驗證分類系統可順利擴充
- [ ] (可選)新增一支 migration 移除 `events` 表
