# Agentic AI Coding Workflow

這套流程要求每次修改同時做到：完成需求、保留既有正確行為，並讓最終結構看起來像新需求從一開始就存在。

AI 容易因 **additive bias** 而新增 class、wrapper、flag 或 compatibility layer，卻不主動整理已失去責任的舊結構。因此 workflow 必須分別驗證 correctness 與 architecture。

---

## Workflow Overview

```text
0. Classify task and choose control mode
1. Define the spec

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

Step 0 先依任務風險決定控制模式與實際路徑；Steps 1–6 再依分類完整執行、縮減、合併或條件式跳過。

---

## Task Workspace

每個 task 使用獨立 Git branch，並在 Step 0 前切換完成。Branch name 不含 `/`、無固定前綴；文件統一放在同名的 `branch_doc/<branch-name>/`。

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

## 0. Classify Task

- **Trivial：** 文件、文字，或已確認不改變 runtime、build、deployment、security semantics 與 observable behavior 的小型機械修改。Dependency、CI、feature flag、build 或 deployment config 無法確認 semantics 不變時，至少分類為 Bounded。
- **Bounded：** 局部 bug fix、單一功能、小型 refactor，或影響清楚且不改變主要架構邊界的工作。
- **Architectural：** 新 subsystem、跨模組行為、資料格式、公開介面、responsibility、ownership，或可能影響多個下游 consumer 的修改。

Trivial 任務使用 Sol 與 Continuous 模式，不建立 routing 文件。Bounded 與 Architectural 任務必須使用 `$workflow-routing`，並在 `branch_doc/<branch-name>/model-routing.md` 記錄 workflow path、control mode、executor、model、context 與 conditional Steps。

Step 0 至少輸出：

```text
Task class:
Control mode:
Workflow path:
Routing artifact: <path or none>
```

執行中發現隱藏複雜度時升級分類並重新 routing。未觸發的 conditional Step 必須說明跳過理由。Architectural 任務若方向與授權已明確，不因分類本身額外要求產品決策。

---

## 1. Spec

Spec 把需求轉成可判斷的完成條件，不預先指定不必要的實作細節。

依任務深度確認：

- 要改變的 observable behavior
- 必須保留的既有行為
- 已知 corner cases
- 可自動驗證的結果
- 可能改變的 responsibility、ownership 與影響範圍

Architectural 任務若存在會實質改變產品行為、架構方向或範圍的未決選擇，取得使用者批准後才能進入 Implementation。

**輸出：** 與任務深度相稱的 acceptance criteria、影響範圍，以及 Architectural 任務的可 review implementation plan。

---

## 2. Implementation

先閱讀相關 call path、tests 與既有 abstraction，再以可控制、可驗證的最小修改完成需求。

- 說明修改哪些 responsibility，而不只列檔名。
- 優先修改現有模型；新增 abstraction 前說明必要性。
- 依測試策略處理新行為與已知 corner cases。
- 不做與需求無關的大型重構。
- 留意 boolean flag、forwarding wrapper、重複 validation、舊 class 與 compatibility branch 等 additive bias 產物。

**輸出：** 能進入 Correctness Gate 的 working implementation 與 focused evidence。

---

## 3. Correctness Gate

Correctness Gate 只回答：**目前的行為是否正確？**

依專案與任務風險執行必要的 build、compile、unit、regression、integration、acceptance、lint、type check 或其他 static checks。綠燈只有在證據能對應需求核心時才算通過。

### Proportional testing

- 可重現 bug、deterministic domain logic、parsing、persistence、state transition、public contract 與可精確描述的 corner case，優先 test-first：先確認測試因正確原因失敗，再做最小修正、驗證並重構。
- Legacy code、UI orchestration、外部 integration boundary，或必須先探索才能確定穩定介面的工作，可以同時或事後補測試，但完成前仍需核心行為證據。
- 純文件、已確認 semantics 不變的機械設定，以及由 compiler、formatter、linter 或 schema tooling 完整保證的規則，不要求人造測試。
- 不得只為形式而測試 mock、實作細節或沒有回歸價值的內容。

人工發現的 bug 與 corner case 應盡量轉成永久 regression test。

**Gate：** 所有必要驗證通過，而且證據能對應 acceptance criteria。

---

## 4. Architecture Gate

在 correctness 已成立後重新問：

> **If the new requirement had existed from the beginning, would we still have designed the system this way?**

依任務深度檢查：

- 受影響的 class / module 是否仍有單一、清楚的責任？
- ownership 搬走後，舊 abstraction 是否只剩轉接或歷史包袱？
- 新舊路徑是否表達同一個 concept？
- 是否出現兩套 state、validation 或 configuration source？
- wrapper、adapter、flag 或 compatibility layer 是否代表長期需要的差異？
- 哪些結構現在可以刪除、合併、改名或搬移？

這一步與 Implementation 分開進行；必要時使用獨立 review context 或 subagent。

**Gate：** 說明應保留與應簡化的結構，以及具體理由。

---

## 5. Simplification

執行 Architecture Gate 中有具體理由的清理，例如刪除失去責任的路徑、合併重複 abstraction、搬移 ownership、取代 compatibility wrapper，或更新反映目前 mental model 的名稱與文件。

簡化的判準是用較少概念理解系統，而不是追求較少行數。刪除前確認 dependency；保留舊結構也要指出具體 dependency。

走過此 Step 不代表一定要修改。沒有值得處理的 findings 時，記錄不需 simplification 的理由，不得為完成形式而製造重構。

若此 Step 修改了 architecture，返回 Step 4，由 Reviewer 以更新後的 repository 與 actual diff 重新執行 Architecture Gate。Recheck 發現新 findings 時再次返回 Step 5；只有 Architecture Gate 通過後才能進入 Step 6。沒有修改時沿用原 Step 4 結論，不另行 recheck。

**輸出：** 完成有理由的簡化並通過 Step 4 recheck，或記錄不修改的理由。

---

## 6. Regression Again

Correctness Gate 後若修改 code、configuration 或其他受驗證內容，重跑依影響範圍與風險判定為必要的驗證，不得沿用修改前的綠燈。Trivial 與 Bounded 沒有 post-gate change 時可以跳過；Architectural 依 Step 0 執行 final regression。

依任務需要確認：

- 必要 build / compile 與影響範圍內測試通過
- 新需求、integration 與 acceptance evidence 通過
- 移除的路徑沒有 dead reference 或漏接 caller
- final diff 與 Spec 一致且沒有無關變更

完整 suite 無法執行時，揭露原因、替代證據與剩餘風險。

---

## Completion Contract

Done 代表適用的 acceptance criteria、build、static checks、behavior evidence、Architecture Gate 與 post-gate regression 都已通過，且未驗證項目與風險已揭露。

依任務規模報告：

- acceptance criteria 狀態
- 實際修改的責任、檔案與行為
- verification commands 與 pass / fail 結果
- Architecture Gate findings
- simplification，或不需簡化的理由
- 未驗證項目、剩餘風險與人工 checks

Trivial 可以只報告修改摘要、相關檢查與 final diff review。Bounded 與 Architectural 必須提供足以對應 acceptance criteria 的證據，不得以「看起來正確」、推測或修改前的綠燈宣稱完成。

Step-gated 模式完成 Step 6 後先提交 completion evidence；取得使用者接受後才能標為 Done。
