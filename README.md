# zhtw-vocab-guard

台灣繁體中文技術詞彙檢查器：掃描 Markdown 文件裡的大陸慣用技術詞，以及 OpenCC 把簡體轉成繁體時留下的二次退化詞，可以只檢查，也可以就地修補。單一 Python 檔，只用標準函式庫，不連網。

> 這是**固定字串詞庫**，不是語義校對工具。通過只代表「沒踩到內建的 33 條規則」，不代表用詞都對。抓不到的東西見下方「已知限制」。

## 它在文件流程的哪一步

```
 寫文件／OpenCC 轉換 ──► --check（CI 或提交前） ──► 有 [ALERT] ──► --fix 就地修補 ──► 再 --check
                              │                                         │
                              ▼                                         ▼
                      結束碼 0 才放行                        反覆替換到固定點，重跑結果不變
```

## 安裝

需要 Python 3.11 以上，不需安裝任何套件。

```bash
git clone https://github.com/SanHsien/zhtw-vocab-guard.git
cd zhtw-vocab-guard
python tools/verify_zhtw_vocab.py --help
```

要在別的 repo 使用，直接用絕對路徑呼叫這支檔案，或把 `tools/verify_zhtw_vocab.py` 複製過去（單檔、無相依）。

## 用法

| 指令 | 用途 |
|------|------|
| `python tools/verify_zhtw_vocab.py --check <路徑>...` | 只檢查。目錄會遞迴收集 `.md`，明確指定的檔案直接處理。命中時每筆印一行 `[ALERT] 檔案:行號` |
| `python tools/verify_zhtw_vocab.py --fix <路徑>...` | 就地修補，保留原檔換行。先檢查全部輸入再動手；多檔寫入沒有交易回滾 |
| `python tools/verify_zhtw_vocab.py --verify-idempotent` | 驗證內建規則的冪等性：修補過的文字再修補一次，結果不變 |

三個選項可以併用，例如 CI 常用的：

```bash
python tools/verify_zhtw_vocab.py --verify-idempotent --check README.md docs
```

結束碼：

| 結束碼 | 意思 |
|--------|------|
| 0 | 請求的操作都通過 |
| 1 | 發現詞彙問題，或冪等樣本失敗 |
| 2 | 引數、輸入或讀寫錯誤：沒給任何操作、路徑不存在、沒有可檢查的檔案、檔案是空的、不是 UTF-8 |

`[REVIEW]` 只是提示人工確認（裸 URL 裡含規則詞時保留原文並提示），不影響結束碼；`[ALERT]` 才會讓檢查失敗。

也可以當模組用：

```python
from verify_zhtw_vocab import apply_zhtw_vocab, scan_degraded_vocab

fixed = apply_zhtw_vocab(text, skip_code=True)   # 回傳修補後文字
hits = scan_degraded_vocab(text)                 # 回傳命中清單（term、replacement、reason、line）
```

## 規則範圍

共 33 條，分兩組，寫在 `tools/verify_zhtw_vocab.py` 開頭的兩個常數：

- **`GLOBAL_VOCAB_RULES`（20 條）**：高確定性的技術複合詞與動作詞。例如 `服務器`、`軟件`、`硬件`、`視頻`、`數據庫`、`環境變量`、`源代碼`、`命令行`、`正則表達式`、`默認值`、`中間件`、`重定向`、`粘貼`、`摁下`。
- **`DEGRADED_AND_CONTEXT_RULES`（13 條）**：OpenCC（`cn` → `tw2`）二次轉換的退化詞，與只在特定片語才成立的語境詞。例如 `連線埠`、`隻影響`、`型別別`、`瞭什麼`、`參考檔案`、`關鍵數位`。

不會被改動的地方（`skip_code=True`，CLI 預設）：

- Markdown 圍欄程式碼區塊，與同一行內長度相同的反引號 run。要在文件裡舉錯誤用詞當反例，就放進反引號。
- 成對的中文引號（`「」`、`『』`）裡的原話：最多 200 字，不跨空白段落或表格欄；錯配的引號不保護。
- URL 與連結目的地：裸 URL、自動連結、行內連結與參考式連結的目的地。連結的顯示文字與標題照常檢查。

少數規則帶有前後字的例外，避免切到別的詞（例如「電視頻道」「巡視頻率」不算命中）。

## 已知限制

- **沒有專名保護**：未加引號的產品名稱若含規則詞，一樣會被改。
- **不判斷語義**：不看詞性、標題、標點，也不判斷整句是否通順。
- **「支持」不會自動換成「支援」**：兩者在台灣都是正確用詞，要靠受詞判斷，目前未收。
- **「數位」只收兩個片語**：台灣的「數位」是 digital 的正確用詞，所以只收「同樣的…」「關鍵…」兩個數字被誤轉的片語，其餘位置的誤轉抓不到；例外字集以外的 digital 複合詞也可能被誤改。
- **前後字例外是字集比對**：仍可能誤排除罕見組合（例如「接收」後面緊接影片一詞的大陸寫法）。
- **Markdown 解析是有限的**：不是完整 CommonMark；跨行的行內程式碼不支援，連結目的地只認同一行。
- **不執行 OpenCC**：只做內建詞庫替換。`--verify-idempotent` 驗的是內建規則與重疊樣本，不是 OpenCC 端到端測試，也不是對任意文字的形式證明。
- **`--fix` 只修用詞**，不保證文件事實正確。

後續要補的項目見 [ROADMAP.md](ROADMAP.md)。

## 文件

- [上游簡體開源專案之零漂移繁體中文 Fork 自動同步規範](docs/zhtw-fork-idempotent-sync-spec.md)：對照上游的同步腳本逐段說明「從上游重建分支、OpenCC 轉換、詞庫修補」的做法，以及本工具的實際契約。

## 開發

```bash
# 不要設 PYTHONUTF8，CI 刻意用系統預設編碼
python -m unittest discover -s tests
python tools/verify_zhtw_vocab.py --verify-idempotent --check README.md AGENTS.md ROADMAP.md docs
```

規範見 [AGENTS.md](AGENTS.md)。

## 致謝與來源

- 詞庫修補加冪等檢查的做法，以及二次退化詞與語境詞的第一批規則，借鑑 Will 保哥 [doggy8088/answer-me-with-html](https://github.com/doggy8088/answer-me-with-html)（`zh-tw` 分支）的 `scripts/zh-tw/vocab.mjs` 與 `check.mjs`。本 repo 是以 Python 重新實作並自行延伸規則，不是上游詞庫的完整複製。
- 本 repo 於 2026-11-19 自 [SanHsien/ai-inbox](https://github.com/SanHsien/ai-inbox) 抽出，保留原檔的 git 歷史。當初的分析筆記與任務紀錄留在 ai-inbox：[保哥繁中 Fork 筆記](https://github.com/SanHsien/ai-inbox/blob/main/notes/2026-11-07-willh-doggy8088-answer-me-with-html-zhtw-fork.md)。

## 授權

[MIT](LICENSE)
