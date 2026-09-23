# 職缺維護系統（GAS 後端）專案交接筆記

給另一個 Claude Code session 接續使用。這份文件整理專案架構、部署方式，
以及已經踩過的雷，避免重複踩坑。

## 專案基本資訊

- 專案性質：「職缺維護系統」（Netlify 前端 + Google Apps Script 後端）
  的後端邏輯，處理職缺送審、薪資補款送審、專案合約、同仁 LINE 綁定等
  表單背後的所有業務邏輯（AI 潤飾文案、權限檢查、寫入 Notion/試算表、
  LINE 推播審核、核准後寄信）。
- 技術棧：Google Apps Script，部署成 Web App 接收表單 POST（`doPost`）
  跟少數查詢/橋接請求（`doGet`）。
- **2026-09-22 才第一次接上 GitHub 版控**：在這之前這份程式碼只存在
  Apps Script 編輯器裡，沒有版本紀錄、Claude Code 也看不到——這次用
  `clasp clone` 把既有專案的程式碼原封不動抓下來推上這個 repo，**不是
  重新建立一個新的 GAS 專案**，`.clasp.json` 的 `scriptId` 對應的還是
  原本正式在用的那個專案。
- 跟這個 repo 互相搭配的另外兩個 repo：`tsaipei-linebot/tsaipeilinebot`
  （材霈平台本體，`/me` 薪資補款紀錄、`/finance` 財務部專區、`/job-listings`
  職缺維護的表單都是材霈平台這邊收，送出後轉手給這個 GAS 專案處理，見
  那個 repo 的 `services/salary_repayment_submit_service.py`／
  `services/job_listing_submit_service.py`／`job_portal_sso.py` 的開頭
  說明）、`tsaipei-linebot/delivery-gas-project`（配送部系統的另一支
  GAS，完全獨立，不要搞混）。

## 檔案結構

| 檔案 | 內容 |
|---|---|
| `程式碼.js` | 共用邏輯：`CONFIG`（指令碼屬性集中讀取）、`doGet`/`doPost` 路由、`isValidAdminApiSecret()`/`isValidAdminApiSecretFromBody()`（共用密鑰驗證，前者給 doGet 網址參數用，後者給 doPost body 用）、同仁 LINE 帳號自動綁定（`EmployeeRegistrationService`） |
| `Project_Salary.js` | 薪資補款：送審（`SalaryWorkflowService.processSalarySubmission`）、LINE 審核核准/退回（`handleSalaryPostback`）、核准後寄信（`EmailService.sendSalaryCompensationReport`）、補寄信（`resendSalaryEmail`，2026-09-22 新增）、財務部批次匯出 PDF（`exportApprovedSalaryPdfsZip`，2026-09-22 新增） |
| `Project_Job.js` | 職缺維護：送審、AI 潤飾文案（含就業服務法禁語過濾）、寫入 Notion、LINE 推播審核 |
| `Project_BatchEnhance.js` | 批次 AI 增強/遠端快取管理 |
| `ProjectWorkflowService.js` | 專案合約送審流程 |

## 部署

- `.github/workflows/clasp-push.yml`：合併到 `main` 後自動 `clasp push`，
  跟 `delivery-gas-project` 同一套做法，需要先設定好 `CLASPRC_JSON`
  這個 repository secret（步驟見 `tsaipeilinebot` 專案 `HANDOFF.md`
  「CI/CD 自動部署」章節，兩邊做法一致）。
- **2026-09-22 確認結果：是固定版本的部署，不是 @HEAD**——比對
  `tsaipeilinebot` 的 `JOB_PORTAL_GAS_WEBAPP_URL` 環境變數（材霈平台
  實際呼叫的網址）跟 `clasp deployments` 的清單，確認材霈平台打的是
  deployment id `AKfycbwi-j_mbnUDRFPKyEvL7arPv9UzHqpJLoNf9xHMOZTIf2yPN-ob5gDyFvwxKU63mIhVIA`
  （清單裡標註「薪資補款：新增PDF存查單附件」那筆）——`delivery-gas-project`
  當初踩過的那個雷（固定版本部署，光 push 不會生效，要另外
  `clasp deploy -i <deployment id>`）這裡真的發生了：9/22 稍早合併的
  「修正薪資補款已核准但沒收到信＋補寄信＋財務部專區」那次，`clasp push`
  CI 顯示成功，但正式環境其實沒有真的更新（因為當時 `CLASPRC_JSON`
  也還沒設定，push 根本沒執行成功，兩個問題疊在一起）。已經在
  `clasp-push.yml` 補上 `clasp deploy -i` 這一步（照抄
  `delivery-gas-project` 的做法），之後合併到 main 都會自動更新到這個
  固定部署，不用再手動處理。

## 指令碼屬性（只列名稱與用途，實際值不寫進 git）

`LINE_CHANNEL_ACCESS_TOKEN`/`LINE_WEBHOOK_SECRET`（這組職缺系統專屬 LINE
官方帳號，跟材霈平台的招募機器人「沛沛」完全獨立）、`ADMIN_API_SECRET`
（保護 `doGet` 管理端查詢，2026-09-22 起也保護 `RESEND_SALARY_EMAIL`／
`EXPORT_SALARY_PDFS` 這兩個 `doPost` 端點——`tsaipeilinebot` 那邊要設定
同一個值的 `JOB_PORTAL_ADMIN_API_SECRET` 環境變數，兩邊密鑰要一致）、
`NOTION_API_KEY`/`NOTION_DATABASE_ID`、`GOOGLE_DRIVE_FOLDER_ID`、
`SPREADSHEET_ID`、`HR_ACCOUNTING_EMAILS`、`ADMIN_LINE_USER_ID`、
`GCP_PROJECT_ID`/`GCP_LOCATION`（Vertex AI）、`PIN_PEPPER`（同仁 LINE
綁定 PIN 碼加鹽雜湊用）。

## 2026-09-22：修正薪資補款「已核准但沒收到信」+ 新增補寄信/財務部批次匯出

**根本原因**：`Project_Salary.js` 的 `SalaryWorkflowService.
handleSalaryPostback()`，主管在 LINE 按核准後，不管
`EmailService.sendSalaryCompensationReport()` 這一步實際成功還是失敗，
回覆給主管/申請人/其他主管的 LINE 訊息永遠都說「已自動寄出」——寄信
失敗（例如收件人清單湊不出有效 email、`GmailApp.sendEmail()` 拋例外）
只印進 `console.error`/`console.warn`，只有主動查「執行項目」才看得到，
沒有人知道要處理。使用者反映「送出後沒收到 mail」才追查發現。

**修正**：
- `EmailService.sendSalaryCompensationReport()` 改成回傳
  `{success, message, recipients}`（找不到有效收件人、`GmailApp.
  sendEmail()` 拋例外，都算失敗並帶上原因），不再是靜默 `return`。
- `handleSalaryPostback()` 的三則 LINE 訊息（回覆審核者、通知申請人、
  同步通知其他主管）都改成依真實結果組文字，失敗時提示可以用「補寄信」
  功能重試。

**新增兩個 `doPost` 端點**（都要驗證 `admin_secret`，不是公開表單）：
- `RESEND_SALARY_EMAIL`：帶 `salary_id`，重新抓這筆紀錄（要是「已核准」
  狀態才會真的補寄）、再呼叫一次 `sendSalaryCompensationReport()`。給
  `tsaipeilinebot` 的 `/me` 薪資補款紀錄頁面「補寄信」按鈕用。
- `EXPORT_SALARY_PDFS`：帶 `start_date`/`end_date`（比對「申請日期」
  欄位），撈出區間內所有已核准紀錄，各自用既有的
  `EmailService.buildSalaryPdfBlob()`（**跟核准信附件同一份排版邏輯，
  沒有另外重刻**）產生 PDF，`Utilities.zip()` 打包成一個 ZIP、base64
  編碼後回傳。給 `tsaipeilinebot` 的 `/finance` 財務部專區批次下載用。

`SalarySheetService` 新增 `_buildRecordFromRow()`（把「試算表一列→完整
紀錄物件」這段 20 幾行的欄位對照抽成共用函式，原本
`updateSalaryReviewStatus()` 裡重複寫一份，之後試算表欄位異動只要改
一個地方）、`getFullRecordById()`（補寄信用）、
`listApprovedRecordsInRange()`（批次匯出用）。

完整規劃討論、跟 `tsaipeilinebot` 那邊的串接細節，見 `tsaipeilinebot`
專案 `HANDOFF.md`「修正薪資補款『已核准但沒收到信』+ 新增『補寄信』＋
『財務部專區』」章節，不重複記在這裡。

這個 repo 目前沒有自動化測試（GAS 專案，跟 `delivery-gas-project` 一樣
沒有寫測試的既有慣例），改動前後都用 `node --check <file>.js` 做語法
檢查，功能是否正確要靠使用者在正式環境（或 Apps Script 編輯器的
「執行」功能）實測確認。

## 薪資補款通知信改由材霈平台寄出（2026-09-23）

### 為什麼要改

2026-09-23 同仁回報「補寄信按了沒有動作」，查出來是
`Exception: 單日叫用下列服務的次數過多：email。`——**Apps Script 的寄信
額度是「每個 Google 帳號每天 100 個收件人」**（這支 script 跑在一般
Gmail 帳號上；Google Workspace 是 1500）。一封補款通知信要同時寄給
財會＋審核主管＋申請人，一封就吃掉 3～5 個額度，所以**每天大約核准
20～25 筆就會撞到上限，撞到之後那天所有通知信都寄不出去**（不只是手動
補寄的那幾筆）。

同一個信箱改走 SMTP 寄，是完全不同的一條額度（一般 Gmail 約 500、
Workspace 約 2000），所以把「寄出去」這一步搬到材霈平台（Cloud Run）
就能立刻拉高上限，之後要換成正規寄信服務也只要改平台那邊。

### 改了什麼（刻意只搬「寄出去」這一步）

**核准流程、寫試算表、產 PDF 存查單、組信件 HTML 全部維持在這支 GAS
不變**，只有最後真正送出去那一下改成呼叫材霈平台：

- `程式碼.js` 的 `CONFIG` 新增 `PLATFORM_MAIL_URL`／`PLATFORM_MAIL_SECRET`
  兩個指令碼屬性。
- `Project_Salary.js` 的 `EmailService` 新增 `sendViaPlatform()`：把
  收件人、主旨、HTML、附件（base64）、內嵌圖片（base64＋content_id）
  POST 到平台的 `/api/job-portal/send-mail`。
- `sendSalaryCompensationReport()` 改成「兩個指令碼屬性都有設定就走平台，
  否則退回原本的 `GmailApp.sendEmail`」——**漏設定不會讓通知信整個斷掉**，
  而且萬一平台那邊有狀況，把指令碼屬性清掉就立刻回到原本的寄法。
- 內嵌圖片（補款佐證照片）照舊用 `cid:salaryProofImg` 顯示在信件內文，
  不是只當附件——平台端有對應處理，信件外觀跟原本一模一樣。

### 上線前要設定的指令碼屬性

Apps Script 專案 → 專案設定 → 指令碼屬性，新增兩筆：

| 屬性名稱 | 值 |
|---|---|
| `PLATFORM_MAIL_URL` | `https://recruitment-bot-412901869672.asia-east1.run.app/api/job-portal/send-mail` |
| `PLATFORM_MAIL_SECRET` | 跟平台 Cloud Run 的 `JOB_PORTAL_MAIL_WEBHOOK_SECRET` 環境變數設成同一個值 |

平台那邊還要設定 SMTP 帳號密碼才會真的寄得出去，完整步驟見
`tsaipeilinebot` 專案 `HANDOFF.md`「薪資補款通知信改由平台用 SMTP 寄」
章節。

改動前後都用 `node --check` 做語法檢查（這個 repo 沒有自動化測試的既有
慣例），實際寄信效果要在正式環境實測確認。
