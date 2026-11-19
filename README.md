# AI 作業自動評分系統（Google Apps Script + Gemini API）

一個基於 **Google Apps Script (GAS)**、**Google 試算表** 與 **Google Gemini API** 打造的無伺服器（Serverless）作業繳交與 AI 自動評分系統。

教師可自訂作業與評分標準，系統會自動串接 Gemini 模型對學生的文字作答與程式碼/純文字檔案進行評分、說明給分理由並提供具體回饋；所有學生上傳的檔案**不落地儲存實體檔案**，僅以純文字形式寫入 Google 試算表保存。

---

## ✨ 功能特色

### 👩‍🏫 教師功能
* **權限控管**：透過指令碼屬性 `TEACHER_EMAILS` 指定教師 Email 清單，支援多位教師共同管理。
* **作業管理**：建立與編輯作業（標題、作業說明、AI 評分標準、滿分設定、開放/關閉繳交狀態）。
* **多維度成績檢視**：
  * **成績總覽（以最後一筆為準）**：自動統計每位學生的繳交次數與最終採計分數。
  * **所有繳交明細**：保留並檢視每位學生歷次上傳的完整內容、檔名、AI 評分與評語。
* **重新評分與匯出**：支援針對單筆作業點擊「重新以 AI 評分」，並可一鍵匯出採計成績 CSV（含 UTF-8 BOM，Excel 開啟不亂碼）。
* **視角切換**：教師可隨時切換至「學生繳交視角」進行測試繳交。

### 👨‍🎓 學生功能
* **彈性作答方式**：支援「文字框直接作答」與「多檔案拖曳/選取上傳」。
* **支援多種純文字與程式碼格式**：`.txt`、`.md`、`.csv`、`.ipynb`、`.py`、`.json`、`.js`、`.ts`、`.html`、`.css`、`.sql`、`.java`、`.c`、`.cpp` 等。
* **多次繳交機制**：同一份作業可重複繳交多次，每次都會產生獨立紀錄與 AI 批改結果，**採計成績以最後一筆為準**。
* **即時 AI 回饋**：送出後即時取得分數、逐項評分理由及具體改進建議。

### 🔐 技術與安全設計
* **檔案不落地**：前端透過 `FileReader` 讀取純文字後送往後端寫入試算表，不使用 Google Drive 儲存檔案。
* **試算表與 API Key 隔離**：主系統以「開發者（我）」身分執行，學生無須取得試算表存取權限，避免偷看他人成績或竄改資料。
* **防範單格字數上限**：自動將上傳檔案文字拆分至 `filesTextContent` 與 `filesTextOverflow`（最高支援 90,000 字元），並針對 `.ipynb` 自動濾除過大的圖片 Base64 輸出。
* **結構化輸出與防注入**：使用 Gemini API `responseSchema` 強制輸出結構化 JSON（`score`、`reasoning`、`feedback`），並於 System Instruction 加入提示詞注入（Prompt Injection）防護，遇到 `429`/`503` 狀態會自動重試一次。
* **三合一登入相容架構**：
  1. **同網域原生登入**（Google Workspace 組織內自動辨識 Email）。
  2. **Email 6 位數驗證碼登入 (OTP)**（內建支援任意外部 `@gmail.com`）。
  3. **一鍵 Google 帳號驗證 (`AuthRelay`)**（選用獨立專案，外部 `@gmail.com` 免收驗證信直接登入）。

---

## 📁 專案檔案結構

```text
├── 主專案（綁定於 Google 試算表，或獨立專案）
│   ├── Code.gs          # 後端 API、試算表讀寫、認證驗證、Gemini API 介接
│   └── Index.html       # 前端響應式介面（教師後台、學生繳交區、多檔純文字讀取）
│
└── AuthRelay 專案（【選用】獨立的 GAS 專案，僅用於外部 Gmail 一鍵 Google SSO）
    └── AuthRelay.gs     # 取得外部使用者 Email 並以 HMAC-SHA256 簽章導回主系統
```

---

## 🚀 安裝與部署步驟

### 步驟一：準備 Google 試算表與主專案
1. 新增一份 Google 試算表（此試算表將作為資料庫，**保持私人不共用給學生**）。
2. 點選試算表上方選單 **「擴充功能」→「Apps Script」**。
3. 將 `Code.gs` 內容貼入預設的程式碼檔案中。
4. 新增一個 HTML 檔案命名為 `Index`（系統會自動存成 `Index.html`），貼入 `Index.html` 完整程式碼並儲存。

### 步驟二：設定主專案「指令碼屬性」
點選 Apps Script 左側 **「專案設定（齒輪圖示）」→ 下方「指令碼屬性」→「新增指令碼屬性」**：

| 屬性名稱 | 必填 | 說明與範例 |
| :--- | :---: | :--- |
| `GEMINI_API_KEY` | ✅ | 前往 [Google AI Studio](https://aistudio.google.com/) 申請的 Gemini API Key |
| `TEACHER_EMAILS` | ✅ | 教師 Email 串列（支援逗號、分號或換行分隔），例：`t1@gmail.com, t2@edu.tw` |
| `GEMINI_MODEL` | 選填 | 預設為 `gemini-3.8-flash`（可依需求改為 `gemini-3.5-flash-lite` 等） |
| `SPREADSHEET_ID` | 選填 | 若綁定試算表可免填；若為獨立腳本可填試算表 ID（未填將自動建立） |
| `AUTH_RELAY_URL` | 選填 | 若有啟用選用的 `AuthRelay` 專案，填入其部署後的 `/exec` 網址 |
| `AUTH_SECRET` | 選填 | 若有啟用 `AuthRelay`，填入與 `AuthRelay` 專案完全相同的自訂密鑰字串 |

### 步驟三：部署主專案為網頁應用程式
1. 點選右上角 **「部署」→「新增部署」**，類型選擇 **「網頁應用程式」**。
2. 權限設定如下：
   * **執行身分 (Execute as)**：**我** *(Me)*
   * **誰可以存取 (Who has access)**：
     * 若全體師生皆為同一學校/機構 Workspace 帳號：選 **「[組織名稱] 內的任何人」**。
     * 若需開放外部個人 `@gmail.com` 帳號登入：選 **「擁有 Google 帳戶的所有人」**。
3. 完成授權與部署後，複製取得的網頁應用程式網址（`https://script.google.com/macros/s/.../exec`），即為主系統網址（`MAIN_APP_URL`）。

---

### 步驟四：【選用】設定 `AuthRelay` 實現外部 Gmail 一鍵登入
> 💡 **說明**：若全體師生都在同一個 Google Workspace 網域，或外部 Gmail 學生可接受使用內建的「Email 6 位數驗證碼」登入，**可完全略過此步驟**。

若希望外部 `@gmail.com` 學生無須收驗證信、點一下按鈕即可用 Google 帳號登入，請另外建立一個**獨立的 GAS 專案**：

1. 前往 [script.google.com](https://script.google.com/) 建立「新專案」，貼入 `AuthRelay.gs` 程式碼。
2. 於該專案的 **「專案設定」→「指令碼屬性」** 新增：
   * `MAIN_APP_URL`：步驟三取得的主專案 `/exec` 網址。
   * `AUTH_SECRET`：自訂一組隨機密鑰字串。
3. 點選 **「部署」→「新增部署」**（類型：網頁應用程式）：
   * **執行身分 (Execute as)**：**存取網頁應用程式的使用者** *(User accessing the web app)*
   * **誰可以存取 (Who has access)**：**擁有 Google 帳戶的所有人** *(Anyone with a Google account)*
4. 部署後取得 `AuthRelay` 的 `/exec` 網址。
5. 回到 **主專案** 的「指令碼屬性」，補填 `AUTH_RELAY_URL`（剛取得的網址）與 `AUTH_SECRET`（與步驟 2 相同）。

---

## 🗄️ Google 試算表資料庫結構

首次載入系統時，`Code.gs` 會自動建立以下兩個工作表與標題列：

### 1. `Assignments`（作業設定）
| 欄位 | 說明 |
| :--- | :--- |
| `assignmentId` | 作業唯一識別碼（`ASG_時間戳記`） |
| `title` | 作業標題 |
| `description` | 作業說明 |
| `rubric` | AI 評分標準 |
| `maxScore` | 滿分（預設 `100`） |
| `isActive` | 是否開放繳交（`TRUE` / `FALSE`） |
| `createdBy` | 建立作業的教師 Email |
| `createdAt` / `updatedAt` | 建立與最後更新時間（ISO 格式） |

### 2. `Submissions`（學生繳交與 AI 評分紀錄）
| 欄位 | 說明 |
| :--- | :--- |
| `submissionId` | 繳交紀錄唯一碼 |
| `assignmentId` | 對應作業 ID |
| `studentEmail` | 學生 Email |
| `attemptNumber` | 該生針對此作業的第 N 次繳交 |
| `submittedAt` | 繳交時間 |
| `textContent` | 文字作答區內容（上限 45,000 字元） |
| `fileNames` | 上傳檔案名稱串列 |
| `filesTextContent` | 上傳檔案合併純文字（前 45,000 字元） |
| `filesTextOverflow` | 超出 45,000 字元時之延伸欄位（最高共 90,000 字元） |
| `aiScore` | AI 評分分數 |
| `aiReasoning` | AI 評分理由說明 |
| `aiFeedback` | AI 回饋與改進建議 |
| `gradingStatus` | 評分狀態（`GRADED`） |
| `modelUsed` | 本次批改使用的 Gemini 模型名稱 |

---

## ⚠️ 常見問題與注意事項

1. **修改程式碼後畫面沒更新怎麼辦？**
   * 在 GAS 中修改 `Code.gs`、`Index.html` 或 `AuthRelay.gs` 後，僅按存檔不會更新已部署的 `/exec` 網址。
   * 必須點選 **「部署」→「管理部署」→ 編輯（鉛筆圖示）→ 版本選擇「建立新版本」→「部署」** 才會生效。
   * *(註：僅修改「指令碼屬性」時不需重新部署，會立即生效。)*
2. **學生點擊 `AuthRelay` 一鍵登入時看到「Google 尚未驗證這個應用程式」？**
   * 因 `AuthRelay` 以「存取的使用者」身分執行，個人帳號首次授權會出現此提示。請引導學生點選 **「進階」→「前往 ...（不安全）」→「允許」**，僅首次需要確認。
3. **檔案上傳限制與 `.ipynb` 處理**
   * 單一檔案限制 2MB 以內，且須為 UTF-8 純文字或程式碼格式（系統會自動阻擋二進位檔案）。
   * 若上傳 Jupyter Notebook (`.ipynb`)，前端會自動移除出力儲存格中的圖片 Base64 編碼，只保留程式碼、Markdown 與文字輸出，以免超過試算表字元上限並節省 Gemini API Token。
4. **Google Apps Script 與 Gemini API 配額**
   * 若使用內建 Email OTP 驗證，免費版個人 Gmail 每日透過 `MailApp` 寄信上限為 100 封（Google Workspace 帳號上限較高）；班級人數較多時建議使用同網域原生登入或啟用 `AuthRelay`。
   * 請留意 Google AI Studio 所設定的 Gemini API 速率限制（RPM/TPM）。
