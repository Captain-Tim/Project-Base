# Agentic AI Coding Workflow

這套流程的目的不是讓 AI 寫出更多 code，而是讓每次修改同時滿足三件事：

1. 新需求真的被完成。
2. 原本正確的行為沒有被破壞。
3. 系統結構看起來像這個需求從一開始就存在，而不是後來加蓋上去。

AI 很擅長快速完成局部任務，但也有明顯的 **additive bias**：遇到新需求時，傾向新增 class、wrapper、flag 或 compatibility layer，而不是主動刪除、合併或重新安排既有責任。結果可能功能完全正常，測試也全數通過，codebase 卻逐步變成「不拆違建，只替違建補強」。

因此，測試只能回答「有沒有壞」，不能單獨回答「現在的結構是否合理」。Agentic workflow 必須同時管理 correctness 與 architecture。

---

## Choose the Workflow Depth

先依影響範圍與風險選擇流程深度。分類是輕量判斷，不是必填表格或額外審批程序；任務性質明顯時，不需要為分類本身打斷工作。

### Trivial

適用於文件或文字修改，以及已確認不改變 runtime、build、deployment、security semantics 或 observable behavior 的小型機械式修改。

- 不要求正式 spec、implementation plan 或 TDD。
- 執行與修改直接相關的 formatter、lint、schema 或 static checks。
- Review 最終 diff，確認沒有無關變更。

Dependency version、CI、feature flag、build config 或 deployment config 即使改法很機械，也可能改變系統行為；只要無法確認上述 semantics 不變，就至少分類為 Bounded。

### Bounded

適用於局部 bug fix、單一功能、小型 refactor，或其他影響範圍清楚且不改變主要架構邊界的工作。

- 先定義 acceptance criteria。
- 新行為或修復的 bug 原則上要有 regression test；無法合理自動化時，說明原因並提供替代驗證證據。
- 先跑 focused tests，再依風險執行必要 regression。
- 完成後進行簡短 Architecture Gate。
- 不要求建立完整的 implementation plan 文件。

### Architectural

適用於新 subsystem、跨模組行為、資料格式、公開介面、responsibility 或 ownership 的改變，以及可能影響多個下游 consumer 的修改。

- 實作前完成足以確認 responsibility、boundary 與主要 tradeoff 的設計分析。
- 若存在會實質改變產品行為或架構方向的未決選擇、需求歧義或範圍擴張，先取得使用者批准；方向與授權已明確時，不因分類本身重複詢問。
- 建立可 review 的 implementation plan，並拆成可獨立驗證的階段。
- 必要時使用獨立 review context 或 subagent。
- 執行完整 Architecture Gate 與 Simplification。若有可行的完整 regression suite，預設完整執行；否則依風險執行必要 regression，並揭露未驗證項目。

發現隱藏複雜度時可以升級分類，不得為了避開必要設計而維持錯誤分類。

---

## Execution Boundaries

### Instruction precedence

遵循執行環境定義的指令優先順序與適用範圍。本文件提供通用 workflow 預設；有效的使用者指示或更具體的專案規則若與其不同，依執行環境判定的較高優先級指示執行。任何指示都不得降低必要的安全與權限邊界。會影響行為、驗證或交付方式的重要偏離，應在結果中揭露。

### Preserve existing working-tree changes

開始修改前先檢查 working tree。除非有明確證據顯示變更由目前任務建立，否則 modified、staged 與 untracked files 一律視為使用者或其他工作的內容。

- 不得 revert、overwrite、delete、move、stage 或順手整理無關的既有變更。
- 必須修改已有變更的檔案時，保留現有內容，只做本次需求所需的最小修改。
- 若既有變更與必要修改直接衝突且無法安全合併，停止並說明重疊位置與風險。
- 限制 formatter 與其他會寫入檔案的工具作用範圍；無法限制時，不得讓它改寫任務外檔案。
- 無關的故障或技術債只記錄與回報，不自行擴大任務修復。

### Proceed or stop

AI 可以自行閱讀 repository、history、call path 與文件，進行範圍內、可逆且沒有未授權外部 side effect 的修改，並執行必要驗證。Spec 足夠明確時，應自行處理局部實作判斷與本次需求內發現的問題，不要每完成一小步就要求確認。

遇到下列情況才停止並詢問：

- 歧義會實質改變產品行為或架構方向。
- 必須擴大到任務外的 subsystem。
- 必須執行不可逆、破壞性、安全敏感或有外部 side effect 的操作。
- 必須 commit、push、merge、publish、release 或改變外部服務狀態，但尚未獲得授權。
- 規格嚴重不足，任何方向都只能猜測。

Architectural 是流程深度，不是停止條件。專案自己的 `AGENTS.md` 可以加入更嚴格的權限限制，其適用方式遵循前述 instruction precedence。

---

## Whole Picture — BFS

先看完整流程，不急著深入每個細節：

```text
1. Spec
   定義要改變什麼，以及什麼叫完成
        ↓
2. Implementation
   以可驗證的最小範圍實作需求
        ↓
3. Correctness Gate
   確認 build、測試與需求行為全部正確
        ↓
4. Architecture Gate
   重新檢查責任、邊界與仍然存在的舊結構
        ↓
5. Simplification
   刪除、合併、搬移或改寫不再合理的部分
        ↓
6. Regression Again
   清理結構後重跑必要驗證，通過才算 Done
```

這六個階段刻意把「把功能做出來」和「把結構整理好」分開。AI 在 Implementation 階段通常會優先追求 task completion；等行為正確後，再以新的系統狀態進行 Architecture Gate，比要求它一開始同時兼顧所有事情更可靠。

### Done 的最低條件

```text
- applicable acceptance criteria are satisfied
- relevant build and static checks pass
- required existing behavior is preserved
- new behavior and known corner cases have valuable tests or other evidence
- obsolete structure has been reviewed in proportion to the change
- required regression passes after simplification
- unverified items and remaining risks are disclosed
```

只套用與任務相關的條件：Trivial 任務不需要人造 acceptance criteria 或測試；Bounded 與 Architectural 任務則必須有足以對應需求的證據。流程不是單向流水線。任何 gate 失敗，都回到最近的合理階段修正，再重新驗證。

---

## 1. Spec

Spec 的工作不是預先設計所有實作細節，而是把「需求」轉成可判斷的完成條件。

Bounded 與 Architectural 任務在實作前應完成下列判斷。Trivial 任務不要求正式 Spec，但仍要理解修改範圍與不得改變的行為。

至少要回答：

- 哪個 observable behavior 要改變？
- 哪些既有行為必須保持不變？
- 有哪些已知 corner cases？
- 哪些結果可由自動測試判定？
- 這次修改可能改變哪個模組的 responsibility 或 ownership？

Spec 應該描述意圖與邊界，不要過早指定一定要新增某個 class 或 wrapper。否則 AI 很容易把暫時想到的解法當成架構要求。

**輸出：** 一組清楚的 acceptance criteria，以及初步影響範圍。

---

## 2. Implementation

Implementation 的目標是用可控制、可驗證的修改完成需求。此階段可以先接受局部、不完美的結構，但不能假裝它就是最終設計。

實作時要求 AI：

- 先閱讀相關 call path、tests 與現有 abstraction。
- 說明它準備修改哪些 responsibility，而不只列出檔名。
- 優先修改現有模型；新增 abstraction 前先解釋其必要性。
- 依比例化測試策略，對新行為與已知 corner cases 補上有回歸價值的測試或替代驗證證據。
- 不順手做與需求無關的大型重構。

要特別留意 additive bias 的典型產物：新的 boolean flag、新的 forwarding wrapper、只剩轉接功能的舊 class、重複的 validation，以及為保留舊流程而疊加的 compatibility branch。

**輸出：** 能完成需求、可進入驗證的 working implementation。

---

## 3. Correctness Gate

Correctness Gate 只回答一個問題：**目前的行為是否正確？**

依專案需要執行：

- build 或 compile
- unit tests
- regression tests
- integration tests
- system / acceptance scenarios
- lint、type check 或其他必要的 static checks

### Proportional testing

下列工作優先使用 test-first：

- 可重現的 bug
- deterministic business 或 domain logic
- parsing、persistence、state transition 或 public contract
- 可以先被測試精確描述的 corner case

基本循環是先寫出能重現需求或 bug 的測試，確認它因正確原因失敗，再做最小修正、確認新舊測試通過，最後視需要重構並重新驗證。

Legacy code、UI orchestration、外部 integration boundary，或必須先探索實際行為才能確定穩定介面的工作，可以在實作同時或之後補測試。完成前仍須提供足以證明核心行為的測試或其他驗證證據。

純文件、已確認不改變 runtime、build、deployment 或 security semantics 的機械式設定，以及已由 compiler、formatter、linter 或 schema tooling 完整保證的規則，不要求人造測試。不得只為滿足形式而測試 mock、實作細節或沒有實際回歸價值的內容。

每次人工發現的 bug 或 corner case，都應盡量轉成永久測試。人的價值是找出「可能哪裡出錯」；自動化測試的價值是確保同一件事不會悄悄再次出錯。

如果測試沒有覆蓋需求核心，只看到綠燈不能算通過。先補足能證明行為的測試，再往下走。

**Gate：** 所有必要驗證通過，而且有證據能對應 Spec 的 acceptance criteria。

---

## 4. Architecture Gate

Architecture Gate 不再問「功能能不能用」，而是問：

> **If the new requirement had existed from the beginning, would we still have designed the system this way?**

也就是把新需求視為系統原始條件，重新檢查目前的結構，而不是接受「舊設計加上一層補丁」作為自然結果。

逐項檢查：

- 每個受影響的 class / module 現在是否仍有單一、清楚的責任？
- ownership 搬走後，舊 abstraction 是否只剩轉接或歷史包袱？
- 新舊路徑是否在表達同一個 concept？
- 是否出現兩套 state、validation 或 configuration source？
- 新增的 wrapper、adapter、flag 是否真的代表長期需要的差異？
- 有哪些東西現在可以刪除、合併、改名或搬到更合理的位置？

這一步最好與 Implementation 分開進行，必要時甚至使用不同的 review context。實作者容易替剛完成的方案辯護；Architecture Gate 需要的是重新建模，而不是證明現有 patch 合理。

Gate 深度應與任務相稱：Trivial 通常只需 final diff review；Bounded 做簡短檢查；Architectural 則完整檢查 responsibility、ownership、boundary 與下游影響。

**Gate：** 能清楚說明哪些結構應保留、哪些需要簡化，以及理由是什麼。

---

## 5. Simplification

Simplification 是 Architecture Gate 的執行階段。目的不是追求抽象上的完美，而是讓最終結構只保留現在真正需要的概念。

常見動作包括：

- 刪除已失去責任的 class、method 或 branch。
- 合併重複的 abstraction 或 validation。
- 把 ownership 搬到實際負責該行為的模組。
- 直接改寫舊模型，取代多層 compatibility wrapper。
- 移除只為過渡實作而存在的 flag、adapter 或 fallback。
- 更新名稱與文件，使其反映現在的 mental model。

簡化不等於「行數越少越好」。真正的判準是：讀者能否用較少的概念理解系統，而且每個保留下來的 abstraction 都有明確存在理由。

刪除前要確認 dependency；保留舊結構時，也要能指出具體 dependency，而不是因為「可能有人會用」。

**輸出：** 行為不變，但 responsibility、ownership 與概念邊界更清楚的 implementation。

---

## 6. Regression Again

Architecture cleanup 會移動責任、刪除路徑，也可能引入新的錯誤。因此 Simplification 完成後，必須再次執行依影響範圍與風險判定為必要的驗證，不能沿用清理前的綠燈。Architectural 任務若有可行的完整 regression suite，預設完整執行；無法執行時要揭露原因、替代證據與剩餘風險。

依任務需要重新確認：

- 必要的 build / compile 通過
- 影響範圍內的既有測試通過
- 新需求測試通過
- integration 與 acceptance scenarios 通過
- 被刪除的舊路徑沒有留下 dead reference 或漏接的 caller
- 最終 diff 與 Spec 一致，沒有無關改動

最後一次 review 同時看兩件事：

```text
Correctness: Did we preserve the required behavior?
Architecture: Is this the structure we would choose today?
```

兩者都通過，任務才算 Done。

---

## Completion Evidence

宣稱完成時，依任務規模報告適用項目：

- acceptance criteria 及各項狀態
- 實際修改的檔案與行為
- 實際執行的 verification commands
- 各項驗證的 pass / fail 結果
- Architecture Gate 的發現
- 已完成的 simplification，或沒有進一步簡化的具體理由
- 尚未驗證的項目、剩餘風險與需要人工執行的 checks

Trivial 任務可以只報告修改摘要、相關檢查與 final diff review。Bounded 與 Architectural 任務應提供足以對應 acceptance criteria 的證據。不得只以「程式看起來正確」、「應該可以」或修改前的綠燈宣稱完成。

---

## Practical Prompt Template for Non-trivial Work

```text
Classify the task as Bounded or Architectural and implement the
requirement according to the acceptance criteria. Match planning and
verification depth to the change's scope and risk.

After the behavior is correct and the required verification passes,
perform a separate architecture review. Assume the new requirement had
existed from the beginning. Identify obsolete responsibilities,
duplicated concepts, compatibility layers, wrappers, flags, or classes
that no longer deserve to exist.

Simplify the structure where justified, then run the full regression suite
again when it is available and proportionate to the risk. Otherwise run
the necessary focused regression and disclose what remains unverified.
Do not declare completion based only on the pre-refactor test result.

Done means:
- the requirement is satisfied
- existing behavior is preserved
- new behavior has valuable tests or other sufficient evidence
- obsolete structure has been reviewed and simplified where appropriate
- the final required verification passes
- unverified items and remaining risks are disclosed
```

這套流程的核心不是要求 AI 一次寫出完美架構，而是建立一個固定節奏：**先證明行為正確，再主動抵抗 additive bias，最後證明清理沒有把功能弄壞。**
