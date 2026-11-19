# AGENTS.md

本 repo 是台灣繁體中文技術詞彙檢查器（單一 Python 檔，只用標準函式庫）。給所有 AI 編碼 agent 與人類維護者看。專案指令檔只放本檔，不建 `CLAUDE.md`、`GEMINI.md`。

## 鐵律

1. **每條規則都要有正例與反例測試。** 新增或修改替換規則，先寫會紅的測試：該改的要改，容易被誤傷的相鄰詞（例如含同樣字串的其他詞）不能動。
2. **修補必須冪等。** `apply_zhtw_vocab(apply_zhtw_vocab(x)) == apply_zhtw_vocab(x)`；新規則要加進 `--verify-idempotent` 能涵蓋的樣本，規則之間不得形成循環。
3. **不確定就不自動改。** 兩岸都通用、或要靠語義判斷的詞（例如「支持」）不整詞替換；只收片語，或只提示 `[REVIEW]`。
4. **錯誤反例放進反引號。** 本 repo 的文件要通過自己的檢查；文件裡舉錯誤用詞時一律寫在行內程式碼或圍欄裡。
5. **只用標準函式庫。** 不加第三方相依；跨平台（Windows／Linux）都要能跑。
6. **工具說明照實寫能力邊界。** 改了偵測或保護行為，同步更新工具開頭的 docstring、README 的「規則範圍」「已知限制」與 ROADMAP。
7. **公開 repo：不放機密、不寫真實的家目錄帳號。**
8. 結束碼語意：0 通過、1 發現詞彙問題或冪等失敗、2 引數／輸入／讀寫錯誤。`[REVIEW]` 不影響結束碼。

## 開發流程

```
寫會紅的測試 ─► 修改規則 ─► 測試轉綠 ─► 冪等與自我檢查 ─► 同步文件 ─► commit
```

- 測試指令：`python -m unittest discover -s tests`。**不設 `PYTHONUTF8`**，要抓得到只在 Windows 預設編碼（cp950／cp1252）才發生的錯誤。
- 自我檢查：`python tools/verify_zhtw_vocab.py --verify-idempotent --check README.md AGENTS.md ROADMAP.md docs`。
- CI 跑 ubuntu-latest 與 windows-latest 的 Python 3.11；本機另以目前的 Python 版本跑一次。
- 測試與 `git commit` 之間用 `&&` 串接，不用 `;`：測試紅燈就不得提交。
- 不改寫已提交的歷史（本 repo 保留 ai-inbox 的原始歷史）。
- GitHub Actions 一律釘 commit SHA，`permissions: contents: read`；唯一例外是 CodeQL 上傳結果需要的 `security-events: write`。
- 檔名 `tools/verify_zhtw_vocab.py` 與模組名 `verify_zhtw_vocab` 是對外介面（其他 repo 以路徑或 `import` 使用），不要改名。
- 若有 `REVIEW.md`，修復其中項目後回註修復 commit 與日期。

## 目錄結構

```
tools/verify_zhtw_vocab.py        檢查器本體（規則、保護區段切分、CLI）
tests/test_verify_zhtw_vocab.py   unittest：規則正反例、保護範圍、冪等、CLI 結束碼
docs/zhtw-fork-idempotent-sync-spec.md   上游繁中 Fork 同步做法對照與本工具契約
.github/workflows/ci.yml       測試與自我檢查（ubuntu／windows，Python 3.11）
.github/workflows/codeql.yml   CodeQL（python、actions）
.github/dependabot.yml         只追 GitHub Actions；freshness、multi-commit 的 waiver 與理由也寫在這裡
README.md   用途、用法、規則範圍、已知限制
ROADMAP.md  後續工作與刻意不做的事（單一真相源）
```
