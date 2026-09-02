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

## Execution Boundaries

這些規則不是可省略的 Step，而是從 repository inspection 開始就適用。

### Instruction precedence

遵循執行環境定義的指令優先順序與適用範圍。本文件提供通用預設；有效的使用者指示或更具體的專案規則若與其不同，依執行環境判定的較高優先級指示執行。任何指示都不得降低安全與權限邊界。影響行為、驗證或交付方式的重要偏離應在結果中揭露。

### Preserve existing working-tree changes

開始修改前檢查 working tree。除非有明確證據顯示變更由目前任務建立，否則 modified、staged 與 untracked files 一律視為使用者或其他工作的內容。

- 不得 revert、overwrite、delete、move、stage 或整理無關的既有變更。
- 修改已有變更的檔案時，保留現有內容，只做本次需求所需的最小修改。
- 若必要修改與既有變更直接衝突且無法安全合併，停止並說明風險。
- 限制 formatter 與其他寫入工具的作用範圍；無法限制時不得讓它改寫任務外檔案。
- 無關故障或技術債只記錄與回報，不自行擴大任務。

### Proceed or stop

AI 可以自行閱讀 repository、history、call path 與文件，進行範圍內、可逆且沒有未授權外部 side effect 的修改，並執行必要驗證。Spec 足夠明確時，不要為局部實作判斷或每個小動作反覆詢問。

遇到下列情況才停止並詢問：

- 歧義會實質改變產品行為或架構方向。
- 必須擴大到任務外的 subsystem。
- 必須執行不可逆、破壞性、安全敏感或有外部 side effect 的操作。
- 必須 commit、push、merge、publish、release 或改變外部服務狀態，但尚未獲得授權。
- 規格嚴重不足，任何方向都只能猜測。

### Workflow transition control

Control mode 管理 Step 0–6 之間的轉換，不管理 Step 內的個別 shell、Git、檔案修改或測試命令。

- **Continuous：** 完成一個 Step 後回報結果，但可直接進入路徑中的下一個 Step。
- **Step-gated：** 完成一個 Step 後提交產出、證據與建議的下一步，然後停止；取得使用者批准後才能繼續。

每個 Step-gated checkpoint 都要先顯示 `Completion Contract` 定義的 Cumulative Workflow Status，再使用：

```text
Step completed:
Outputs and evidence:
Next step:
Status: Waiting for approval
```

Workflow Step 的批准只允許階段轉換，不會授權 destructive operations、version-control 或 release actions、外部 side effect 或範圍擴張。若 gate 失敗，先報告 failure evidence 與建議返回的 Step；Step-gated 模式取得批准後才能返回修正。

各任務深度的預設模式由 Step 0 矩陣定義；使用者或專案規則可以指定更嚴格的模式。

---

## 0. Choose Workflow Depth and Control Mode

先分類任務，再依下表排出實際路徑：

| Stage | Trivial | Bounded | Architectural |
|---|---|---|---|
| Control | Continuous | Step 0 後選擇；預設 Step-gated | Step-gated |
| 1. Spec | 隱含確認範圍 | Lightweight acceptance criteria | Full spec、design 與 plan |
| 2. Implementation | Minimal | Scoped | Staged |
| 3. Correctness | Relevant deterministic checks | Focused tests + risk-based regression | Comprehensive verification |
| 4. Architecture | Final diff review | Short gate | Full gate |
| 5. Simplification | 發現並修正問題時 | 有合理 findings 時 | 處理所有 findings，可結論為不需修改 |
| 6. Regression | Step 3 後有修改時 | Step 3 後有修改時 | Final regression |

### Model routing

> **Model selection snapshot — 2026-09-02（Asia/Taipei）：** 下列型號是基於目前可用模型與能力認知的暫定選擇，不是永久規則。模型能力、工具整合、供應狀態、價格與限制變動快速，至少每月或在新模型／重大版本發布後，以代表性任務重新評估。Implementer／Reviewer 的角色分離原則可以保留，具體型號應隨評估結果更新。

品質優先且可使用 Codex 與 Claude Code 時，依下表分配模型：

| Step | Trivial | 普通 Bounded | 高風險 Bounded | Architectural |
|---|---|---|---|---|
| **Step 0 — 分類** | Sol | Sol | Sol | Sol |
| **Step 1 — Spec** | Sol | Sol | Sol | Sol |
| **Step 2 — Implementation** | Sol | Opus 5 | Opus 5 | Opus 5 |
| **Step 3 — Correctness** | Sol | Sol（新對話） | Sol（新對話） | Sol（新對話） |
| **Step 4 — Architecture** | Sol | Sol | Sol | Sol |
| **Step 5 — Simplification** | Sol* | Opus 5* | Opus 5* | Opus 5* |
| **Step 4 — Recheck** | Sol* | Sol* | Sol* | Sol* |
| **Step 6 — Regression** | Sol* | Sol* | Sol* | Sol |

- **`*`：** 有修改才執行。

- **新對話：** Reviewer 不延續 Step 0–1 的對話，只讀 `spec.md`、repository、actual diff 與 tests，先獨立形成 correctness 與 architecture 判斷。
- **高風險 Bounded：** 模型 routing 的子分類，不改變 workflow depth；適用於 persistence、state transition、public contract、security 或難以回歸的行為。
- **Conditional Step：** 未觸發時仍須說明 skip reason。

### Classification

- **Trivial：** 文件、文字，或已確認不改變 runtime、build、deployment、security semantics 與 observable behavior 的小型機械修改。Dependency、CI、feature flag、build 或 deployment config 無法確認 semantics 不變時，至少分類為 Bounded。
- **Bounded：** 局部 bug fix、單一功能、小型 refactor，或影響清楚且不改變主要架構邊界的工作。
- **Architectural：** 新 subsystem、跨模組行為、資料格式、公開介面、responsibility、ownership，或可能影響多個下游 consumer 的修改。

Step 0 至少輸出：

```text
Task class:
Workflow path:
Control mode:
Known requirements and open questions:
Affected responsibilities:
Verification strategy:
Conditional steps:
```

Blocking behavior 依矩陣的 Control row 執行。Architectural checkpoint 是 workflow transition control；方向與授權已明確時，不因分類本身額外要求產品決策。

執行中發現隱藏複雜度時可以升級分類，不得為避開必要設計而維持錯誤分類。未觸發的 conditional Step 必須說明跳過理由，不得假裝執行。

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

### Status markers

回報多項 checks、acceptance criteria 或 workflow progress 時使用：

- 🟢 Passed，或已完成且已核准
- 🟡 Current step completed and waiting for approval
- 🟠 Concern、partial verification 或 unverified
- 🔴 Failed 或 blocked
- ⚪ Pending、conditionally skipped 或 not applicable；標明具體狀態或理由

每個 🟠、🔴 或 ⚪ 項目都要附簡短說明。不要依賴可能無法一致顯示的文字顏色或 HTML styling。

### Cumulative Workflow Status

每個 checkpoint 先列出完整 workflow 的累積狀態，讓使用者能快速看到整體進度。已核准的舊 Step 只保留一行結果，不重複 evidence；只有目前 Step 在下方展開完整產出與證據。使用者批准後，下一次回報把該 Step 從 🟡 更新為 🟢。

```text
Workflow progress:
- 🟢 Step 0 — <one-line approved result>
- 🟢 Step 1 — <one-line approved result>
- 🟡 Step 2 — <current result; waiting for approval>
- ⚪ Step 3 — Pending
- ⚪ Step 4 — Pending
- ⚪ Step 5 — Conditional
- ⚪ Step 6 — Conditional

Current checkpoint: Step 2
<current step outputs and evidence only>

Next step: Step 3 — Correctness Gate
Status: Waiting for approval
```

Gate 失敗時，將目前 Step 標為 🔴，保留已核准 Step 的 🟢 摘要，並把尚未進入的 Steps 保持為 ⚪。Conditional Step 若不需要執行，使用 ⚪ 並簡述 skip reason。
