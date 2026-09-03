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
Done
```

Step 1 先釐清需求並決定控制模式與實際路徑；Steps 2–6 再依分類完整執行、縮減、合併或條件式跳過。

---

## Task Workspace

每個 task 使用獨立 Git branch，並在 Step 1 前切換完成。Branch name 不含 `/`、無固定前綴；文件統一放在同名的 `branch_doc/<branch-name>/`。

```text
auth-refresh → branch_doc/auth-refresh/
```

---

## Execution Boundaries

- 依指令優先順序執行，不降低安全與權限邊界；影響行為、驗證或交付的重要偏離必須揭露。
- 修改前檢查 working tree。除非確認由本任務建立，既有 modified、staged 與 untracked files 一律視為他人內容，不得改寫、移除或 stage。
- 寫入工具只能作用於任務範圍。與既有變更衝突時停止；無關故障只回報，不擴大任務。

### When to stop

任何 Step 遇到下列情況都停止並詢問：

- 規格不足，或歧義會改變產品行為或架構。
- 必須擴大任務範圍。
- 必須進行尚未授權的破壞性、安全敏感、version-control／release 或外部 side effect 操作。

### Control and reporting

- **Continuous：** 回報 Step 結果後直接繼續。
- **Step-gated：** 完成一個 Step 後提交產出、證據與建議的下一步，然後停止；取得使用者批准後才能繼續。


Checkpoint 狀態：🟢 passed／已核准、🟡 等待核准、🟠 有風險或未驗證、🔴 failed／blocked、⚪ pending／有理由地跳過。🟠、🔴、⚪ 必須說明理由。

```text
Workflow progress: <all Steps; one line each>
Current: <Step and evidence>
Next: <Step>
Status: Waiting for approval
```

已核准 Step 只保留一行；Gate 失敗時回報證據與建議返回的 Step，核准後才修正。

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
  - Trivial 使用 Sol 與 Continuous 模式；Bounded 與 Architectural 使用 `$workflow-routing` 決定 workflow path 與 model routing。
- **Output：** Trivial 確認 scope；Bounded 與 Architectural 建立 `spec.md` 與 `model-routing.md`。
- **Exit criteria：** Scope、acceptance criteria、task class 與 routing 均已明確，且會實質改變產品行為或範圍的未決選擇已取得使用者批准。
- **Next：** 進入 Step 2；後續發現隱藏複雜度時返回 Step 1，更新 Spec、分類與 routing。

---

## 2. Implementation

- **Purpose：** 以可控制、可驗證的最小修改完成 Spec。
- **Input：** Step 1 產出、相關 call path、tests 與既有 abstraction。
- **Actions：**
  - Architectural 任務先決定 responsibility、ownership、affected components 與 change sequence；需要使用者決定的架構方向須在修改前取得批准。
  - 優先修改現有模型；新增 abstraction 前說明必要性。
  - 處理 acceptance criteria 與已知邊界，不做無關重構，並留意 wrapper、flag、重複 validation 等 additive bias。
- **Output：** Working implementation、actual diff 與 focused evidence。
- **Exit criteria：** 實作涵蓋指定 scope，且已準備好接受獨立驗證。
- **Next：** 進入 Step 3；Correctness Gate 失敗時返回 Step 2。

---

## 3. Correctness Gate

- **Purpose：** 回答「目前的行為是否正確？」
- **Input：** Step 1 產出、repository、actual diff 與相關 tests。
- **Actions：** 依專案與風險執行必要的 build、compile、unit、regression、integration、acceptance、lint、type check 或其他 static checks：
  - 可重現 bug、deterministic domain logic、parsing、persistence、state transition、public contract 與可精確描述的 corner case，優先 test-first：先確認測試因正確原因失敗，再做最小修正、驗證並重構。
  - Legacy code、UI orchestration、外部 integration boundary，或必須先探索才能確定穩定介面的工作，可以同時或事後補測試，但完成前仍需核心行為證據。
  - 純文件、已確認 semantics 不變的機械設定，以及由 compiler、formatter、linter 或 schema tooling 完整保證的規則，不要求人造測試。
  - 不得只為形式而測試 mock、實作細節或沒有回歸價值的內容。
  - 人工發現的 bug 與 corner case 應盡量轉成永久 regression test。
- **Output：** Verification commands、結果與能對應 acceptance criteria 的證據。
- **Exit criteria：** 所有必要驗證通過，而且證據能對應 acceptance criteria。
- **Next：** 通過後進入 Step 4；失敗時返回 Step 2。

---

## 4. Architecture Gate

- **Purpose：** 在 correctness 成立後回答：「If the new requirement had existed from the beginning, would we still have designed the system this way?」
- **Input：** Step 1 產出、通過的 Step 3 證據、repository 與 actual diff。
- **Actions：** 依任務深度、並與 Implementation 分開檢查；必要時使用獨立 review context 或 subagent：
  - 受影響的 class / module 是否仍有單一、清楚的責任？
  - ownership 搬走後，舊 abstraction 是否只剩轉接或歷史包袱？
  - 新舊路徑是否表達同一個 concept？
  - 是否出現兩套 state、validation 或 configuration source？
  - wrapper、adapter、flag 或 compatibility layer 是否代表長期需要的差異？
  - 哪些結構現在可以刪除、合併、改名或搬移？
- **Output：** 應保留與應簡化的結構及具體理由。
- **Exit criteria：** 受影響的 responsibility、ownership 與結構均已評估，findings 明確或確認沒有 findings。
- **Next：** 有 findings 時進入 Step 5；否則說明跳過理由並進入 Step 6。

---

## 5. Simplification

- **Purpose：** 處理 Step 4 findings，以較少概念表達最終 architecture。
- **Input：** Step 4 findings、Step 1 產出、repository 與 actual diff。
- **Actions：** 刪除失去責任的路徑、合併重複 abstraction、搬移 ownership、取代 compatibility wrapper，或更新名稱與文件。簡化追求較少概念而非較少行數；刪除或保留舊結構都須確認具體 dependency。
- **Output：** Simplification changes，或保留 finding 的具體理由。
- **Exit criteria：** 所有 findings 已處理，或有具體 dependency 支持保留現狀。
- **Next：** 修改 architecture 時返回 Step 4 recheck；未修改時進入 Step 6。

---

## 6. Final Regression

- **Purpose：** 確認 Step 3 的 correctness 在後續修改後仍成立，且 final diff 可交付。
- **Input：** Step 3 結果、Step 4–5 結果、final diff 與 Step 1 產出。
- **Actions：** Step 3 後若修改受驗證內容，重跑受影響的 Step 3 checks；另確認沒有 dead reference、漏接 caller 或與 Spec 無關的變更。Architectural 必須執行 final regression；Trivial 與 Bounded 沒有 post-gate change 時可以跳過。
- **Output：** Final verification 結果；跳過或無法執行的檢查須附理由、替代證據與剩餘風險。
- **Exit criteria：** 所有必要檢查通過，且 final diff 與 Step 1 產出一致。
- **Next：** 提交 Completion Contract 所需的 completion evidence。

---

## Completion Contract

Done 代表 acceptance criteria 與所有適用的 Steps 已通過，且未驗證項目與風險已揭露。

完成報告包含修改的 responsibility／behavior、驗證結果、architecture／simplification 結論與剩餘風險。Trivial 可以簡短報告；Bounded 與 Architectural 的證據必須能對應 acceptance criteria。

Step-gated 模式完成 Step 6 後先提交 completion evidence；取得使用者接受後才能標為 Done。
