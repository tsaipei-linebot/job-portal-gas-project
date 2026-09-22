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
- **2026-09-22 尚未確認**：這個專案的 Web App 部署是「@HEAD」還是「固定
  版本」——`delivery-gas-project` 當初踩過「固定版本部署，光 push 不會
  生效，要另外 `clasp deploy -i <deployment id>`」這個雷。使用者需要在
  有登入這個 Apps Script 專案權限的環境執行 `clasp deployments` 確認，
  如果是固定版本，要回來請 Claude 在 `clasp-push.yml` 補上對應的
  `clasp deploy -i ...` 步驟，不然以後 push 完 CI 顯示成功，但正式環境
  其實沒有真的更新。

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
