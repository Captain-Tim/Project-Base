# Agentic AI Coding Workflow

這套流程要求每次修改同時做到：完成需求、保留既有正確行為，並讓最終結構看起來像新需求從一開始就存在。

AI 容易因 **additive bias** 而新增 class、wrapper、flag 或 compatibility layer，卻不主動整理已失去責任的舊結構。因此 workflow 必須分別驗證 correctness 與 architecture。

---

## Workflow Overview

```text
1. Define the task, workflow, and completion criteria

repeat
    2. Implement
    3. Run the Correctness Gate
until correctness passes

repeat
    4. Run the Architecture Gate
    5. Simplify when findings exist
until simplification makes no architecture change

6. Run required regression checks
7. Summarize the workflow
Done
```

Step 1 先釐清需求並決定控制模式與實際路徑。Steps 2–7 再依分類完整執行、縮減、合併或條件式跳過。

---

## Task Workspace

每個 task 使用獨立 Git branch，並在 Step 1 前切換完成。Branch name 不含 `/`、無固定前綴。文件統一放在同名的 `branch_doc/<branch-name>/`。

```text
auth-refresh → branch_doc/auth-refresh/
```

---

## Execution Boundaries

- 依指令優先順序執行，不降低安全與權限邊界。影響行為、驗證或交付的重要偏離必須揭露。
- 修改前檢查 working tree。除非確認由本任務建立，既有 modified、staged 與 untracked files 一律視為他人內容，不得改寫、移除或 stage。
- 寫入工具只能作用於任務範圍。與既有變更衝突時停止。無關故障只回報，不擴大任務。

### When to stop

任何 Step 遇到下列情況都停止並詢問：

- 規格不足，或歧義會改變產品行為或架構。
- 必須擴大任務範圍。
- 必須進行尚未授權的破壞性、安全敏感、version-control／release 或外部 side effect 操作。

### Control and reporting

- **Continuous：** 回報 Step 結果後直接繼續。
- **Step-gated：** 完成一個 Step 後提交產出、證據與建議的下一步，然後停止。取得使用者批准後才能繼續。


Checkpoint 狀態：🟢 passed／已核准、🟡 等待核准、🟠 有風險或未驗證、🔴 failed／blocked、⚪ pending／有理由地跳過。🟠、🔴、⚪ 必須說明理由。

```text
Workflow progress: <all Steps, one line each>
Current: <Step and evidence>
Next: <Step>
Status: Waiting for approval
```

已核准 Step 只保留一行。Gate 失敗時回報證據與建議返回的 Step，核准後才修正。

Bounded 與 Architectural 使用兩份跨 Step 文件：目前 executor 在每個 checkpoint 更新 `status.md` 的 Step 狀態、摘要與 artifact。收到使用者意見時立即按 Step 追加至 `user-feedback.md`。該文件不得寫入 executor 自己的 findings，也不得改寫既有內容。

---

## 1. Define Task

- **Purpose：** 釐清需求、決定 workflow depth，並定義完成條件。
- **Input：** 使用者需求、相關 context。
- **Actions：**
  - 釐清要改變的 observable behavior、已知邊界與必須保留的行為。
  - 確認影響範圍，並將需求整理成 acceptance criteria。
  - 依影響分類：
    - **Trivial：** 文件、文字，或已確認不改變 runtime、build、deployment、security semantics 與 observable behavior 的小型機械修改。Dependency、CI、feature flag、build 或 deployment config 無法確認 semantics 不變時，至少分類為 Bounded。
    - **Bounded：** 局部 bug fix、單一功能、小型 refactor，或影響清楚且不改變主要架構邊界的工作。
    - **Architectural：** 新 subsystem、跨模組行為、資料格式、公開介面、responsibility、ownership，或可能影響多個下游 consumer 的修改。
  - Trivial 使用 Sol 與 Continuous 模式，只跑 Step 1 確認範圍、Step 2 修改與自我驗證、Step 7 簡短回報，跳過 Step 3–6 的獨立驗證。Bounded 與 Architectural 使用 `$workflow-routing` 決定 workflow path 與 model routing。
- **Output：** Trivial 確認 scope。Bounded 與 Architectural 建立 `spec.md`、`model-routing.md`、`status.md` 與 `user-feedback.md`。
- **Exit criteria：** Scope、acceptance criteria、task class 與 routing 均已明確，且會實質改變產品行為或範圍的未決選擇已取得使用者批准。
- **Next：** 進入 Step 2。後續發現隱藏複雜度時返回 Step 1，更新 Spec、分類與 routing。

---

## 2. Implementation

- **Purpose：** 以範圍受控、可驗證的修改完成 Spec。
- **Input：** Step 1 產出、repository，以及 Step 3 退回時既有的 `verification.md`。
- **Actions：**
  - 閱讀相關 call path、tests 與既有 abstraction。
  - Architectural 任務先決定 responsibility、ownership、affected components 與 change sequence。需要使用者決定的架構方向須在修改前取得批准。
  - 處理 acceptance criteria 與已知邊界，不做無關重構，並留意 wrapper、flag、重複 validation 等 additive bias。
  - 可穩定重現或精確描述的行為優先 test-first。人工發現的 bug 與 boundary case 應加入 regression test。
  - Trivial 由 implementer 在此 Step 逐項確認 acceptance criteria、檢查 actual diff，並執行適用的輕量檢查；無法驗證時在 completion summary 說明理由與剩餘風險。
  - `verification.md` 在此 Step 為唯讀，不得建立或修改。
- **Output：** Working-tree changes，以及依 acceptance criteria 整理的 implementation summary。Actual diff 由 repository 取得。
- **Exit criteria：** 實作涵蓋指定 scope，且已準備好接受獨立驗證。
- **Next：** Trivial 進入 Step 7。Bounded 與 Architectural 進入 Step 3；Correctness Gate 失敗時返回 Step 2。

---

## 3. Correctness Gate

- **Purpose：** 獨立確認 Spec 與 implementation 是否一致。
- **Input：** Step 1 產出、repository、actual diff 與相關 tests。
- **Actions：**
  - 逐項比對 acceptance criteria、observable behavior、implementation 與 evidence。
  - 發現不一致或證據不足時，記錄未驗證項目或 finding。
  - Repository code 與 tests 在此 Step 為唯讀。需要修改時記錄 finding 並返回 Step 2。
- **Output：** 由 Step 3 executor 建立或更新的 `verification.md`，逐項記錄 alignment 結論、evidence 與 findings。
- **Exit criteria：** 每項 acceptance criterion 都有充分 evidence。
- **Next：** 通過後進入 Step 4。失敗時返回 Step 2。

---

## 4. Architecture Gate

- **Purpose：** 在 correctness 成立後回答：「如果這項新需求從一開始就存在，我們仍會把系統設計成現在這樣嗎？」
- **Input：** `spec.md`、`verification.md`、repository 與 actual diff。
- **Actions：** 依任務深度、並與 Implementation 分開檢查：
  - 受影響的 class / module 是否仍有單一、清楚的責任？
  - ownership 搬走後，舊 abstraction 是否只剩轉接或歷史包袱？
  - 新舊路徑是否表達同一個 concept？
  - 是否出現兩套 state、validation 或 configuration source？
  - wrapper、adapter、flag 或 compatibility layer 是否代表長期需要的差異？
  - 哪些結構現在可以刪除、合併、改名或搬移？
  - Repository code 與 tests 在此 Step 為唯讀。需要修改時記錄 finding 並交由 Step 5 處理。
- **Output：** 由 Step 4 executor 建立或更新的 `architecture-review.md`，記錄應保留與應簡化的結構及具體理由。
- **Exit criteria：** 受影響的 responsibility、ownership 與結構均已評估，findings 明確或確認沒有 findings。
- **Next：** 有 findings 時進入 Step 5，否則說明跳過理由並進入 Step 6。

---

## 5. Simplification

- **Purpose：** 處理 Step 4 findings，以較少概念表達最終 architecture。
- **Input：** `architecture-review.md`、`spec.md`、repository 與 actual diff。
- **Actions：** 逐項處理 findings，在維持 Spec 的前提下移除不必要結構。簡化追求較少概念而非較少行數。刪除或保留結構前確認具體 dependency。`architecture-review.md` 在此 Step 為唯讀。
- **Output：** Working-tree changes，以及每項 finding 的處理結果或不修改理由。
- **Exit criteria：** 所有 findings 已處理，或有具體理由支持不修改。
- **Next：** 有修改時返回 Step 4 recheck。沒有修改且理由成立時進入 Step 6。

---

## 6. Final Regression

- **Purpose：** 確認 Step 3 的 correctness 在後續修改後仍成立，且 final diff 可交付。
- **Input：** `spec.md`、`verification.md`、`architecture-review.md`、repository 與 final diff。
- **Actions：** Step 3 後若修改受驗證內容，重新確認受影響的 acceptance criteria。另確認沒有 dead reference、漏接 caller 或與 Spec 無關的變更。Architectural 必須執行 final regression。Trivial 與 Bounded 沒有 post-gate change 時可以跳過。
- **Output：** 執行時由 Step 6 executor 建立或更新 `regression.md`。跳過或無法驗證的項目須附理由、替代證據與剩餘風險。
- **Exit criteria：** 受影響的 acceptance criteria 仍有充分 evidence，且 final diff 與 Spec 一致、沒有無關變更。
- **Next：** 進入 Step 7。

---

## 7. Completion Report

- **Purpose：** 彙整 workflow 結果與 feedback，完成任務文件。
- **Input：** 各適用 Step 產生的文件、`status.md`、`user-feedback.md` 與 final diff。
- **Actions：** 建立或更新 `final-report.md`，以 high-level 摘要逐 Step 記錄結果、evidence、跳過理由、剩餘風險與使用者 feedback。原始 feedback 保留在 `user-feedback.md`。Trivial 不建 `final-report.md`，直接向使用者簡短回報。
- **Output：** Bounded 與 Architectural 產出 `final-report.md` 與 completion summary。Trivial 只提供 completion summary。
- **Exit criteria：** 報告內容與 Input evidence 一致，且 Bounded 與 Architectural 的 `final-report.md` 已完成。
- **Next：** Done。
