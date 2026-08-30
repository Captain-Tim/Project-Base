# Agentic AI Coding Workflow

這套流程的目的不是讓 AI 寫出更多 code，而是讓每次修改同時滿足三件事：

1. 新需求真的被完成。
2. 原本正確的行為沒有被破壞。
3. 系統結構看起來像這個需求從一開始就存在，而不是後來加蓋上去。

AI 很擅長快速完成局部任務，但也有明顯的 **additive bias**：遇到新需求時，傾向新增 class、wrapper、flag 或 compatibility layer，而不是主動刪除、合併或重新安排既有責任。結果可能功能完全正常，測試也全數通過，codebase 卻逐步變成「不拆違建，只替違建補強」。

因此，測試只能回答「有沒有壞」，不能單獨回答「現在的結構是否合理」。Agentic workflow 必須同時管理 correctness 與 architecture。

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
   清理結構後重跑完整驗證，通過才算 Done
```

這六個階段刻意把「把功能做出來」和「把結構整理好」分開。AI 在 Implementation 階段通常會優先追求 task completion；等行為正確後，再以新的系統狀態進行 Architecture Gate，比要求它一開始同時兼顧所有事情更可靠。

### Done 的最低條件

```text
- requirement is satisfied
- build passes
- existing tests pass
- new behavior has tests
- known corner cases become regression tests
- obsolete structure has been reviewed
- simplification does not break behavior
```

流程不是單向流水線。任何 gate 失敗，都回到最近的合理階段修正，再重新驗證。

---

## 1. Spec

Spec 的工作不是預先設計所有實作細節，而是把「需求」轉成可判斷的完成條件。

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
- 對新行為與已知 corner cases 補上測試。
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

Architecture cleanup 會移動責任、刪除路徑，也可能引入新的錯誤。因此 Simplification 完成後，必須再次執行完整驗證，不能沿用清理前的綠燈。

至少重新確認：

- build / compile 通過
- 所有既有測試通過
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

## Practical Prompt Template

```text
Implement the requirement according to the acceptance criteria.

After the behavior is correct and the tests pass, perform a separate
architecture review. Assume the new requirement had existed from the
beginning. Identify obsolete responsibilities, duplicated concepts,
compatibility layers, wrappers, flags, or classes that no longer deserve
to exist.

Simplify the structure where justified, then run the full regression suite
again. Do not declare completion based only on the pre-refactor test result.

Done means:
- the requirement is satisfied
- existing behavior is preserved
- new behavior is tested
- obsolete structure has been reviewed and simplified where appropriate
- the final build and full regression suite pass
```

這套流程的核心不是要求 AI 一次寫出完美架構，而是建立一個固定節奏：**先證明行為正確，再主動抵抗 additive bias，最後證明清理沒有把功能弄壞。**
