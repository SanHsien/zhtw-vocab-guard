# zhtw-vocab-guard

台灣繁體中文技術詞彙檢查器：掃描 Markdown 文件裡的大陸慣用技術詞，以及 OpenCC 把簡體轉成繁體時留下的二次退化詞，可以只檢查，也可以就地修補。單一 Python 檔，只用標準函式庫，不連網。

> 這是**固定字串詞庫**，不是語義校對工具。通過只代表「沒踩到內建的 34 條規則」，不代表用詞都對。抓不到的東西見下方「已知限制」。

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
| `python tools/verify_zhtw_vocab.py --fix <路徑>...` | 就地修補，保留原檔換行。先檢查全部輸入、算出全部改寫並確認要改的檔都可寫，才開始寫；每個檔以暫存檔加 `os.replace` 原子替換，寫到一半失敗不會留下截斷的檔。多檔之間沒有交易回滾：第 N 個檔替換失敗時，前面已替換的檔維持修補後內容 |
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
| 2 | 引數、輸入或讀寫錯誤：沒給任何操作、路徑不存在、沒有可檢查的檔案、檔案是空的、不是 UTF-8（錯誤訊息會指出檔名）、`--fix` 的目標不可寫 |

`[REVIEW]` 只是提示人工確認（裸 URL 裡含規則詞時保留原文並提示），不影響結束碼；`[ALERT]` 才會讓檢查失敗。

也可以當模組用：

```python
from verify_zhtw_vocab import apply_zhtw_vocab, scan_degraded_vocab

fixed = apply_zhtw_vocab(text, skip_code=True)   # 回傳修補後文字
hits = scan_degraded_vocab(text)                 # 回傳命中清單（term、replacement、reason、line）
```

## 規則範圍

共 34 條，分兩組，寫在 `tools/verify_zhtw_vocab.py` 開頭的兩個常數：

- **`GLOBAL_VOCAB_RULES`（21 條）**：高確定性的技術複合詞與動作詞。例如 `服務器`、`軟件`、`硬件`、`視頻`、`數據庫`、`環境變量`、`開源代碼`、`源代碼`、`命令行`、`正則表達式`、`默認值`、`中間件`、`重定向`、`粘貼`、`摁下`。
- **`DEGRADED_AND_CONTEXT_RULES`（13 條）**：OpenCC（`cn` → `tw2`）二次轉換的退化詞，與只在特定片語才成立的語境詞。例如 `連線埠`、`隻影響`、`型別別`、`瞭什麼`、`參考檔案`、`關鍵數位`。

不會被改動的地方（`skip_code=True`，CLI 預設）：

- Markdown 圍欄程式碼區塊，與同一行內長度相同的反引號 run。要在文件裡舉錯誤用詞當反例，就放進反引號。圍欄前面可以有 BOM、空白或 Tab 縮排（清單內的圍欄）、引用標記 `>` 與清單標記。
- 成對的中文引號（`「」`、`『』`）裡的原話：最多 200 字，不跨空白段落或表格欄；錯配的引號不保護。
- URL 與連結目的地：裸 URL、自動連結、行內連結與參考式連結的目的地。連結的顯示文字與標題照常檢查。

少數規則帶有前後字的例外，避免切到別的詞（例如「電視頻道」「巡視頻率」「型別別名」「資源代碼」「命令行為」「明瞭什麼」不算命中；「開源代碼」改成「開源程式碼」，不會切成「開原始碼」）。

## 已知限制

- **沒有專名保護**：未加引號的產品名稱若含規則詞，一樣會被改。
- **不判斷語義**：不看詞性、標題、標點，也不判斷整句是否通順。
- **「支持」不會自動換成「支援」**：兩者在台灣都是正確用詞，要靠受詞判斷，目前未收。
- **「數位」只收兩個片語**：台灣的「數位」是 digital 的正確用詞，所以只收「同樣的…」「關鍵…」兩個數字被誤轉的片語，其餘位置的誤轉抓不到；例外字集以外的 digital 複合詞也可能被誤改。
- **前後字例外是字集比對**：仍可能誤排除罕見組合（例如「接收」後面緊接影片一詞的大陸寫法）。
- **Markdown 解析是有限的**：不是完整 CommonMark；跨行的行內程式碼不支援，連結目的地只認同一行。四格縮排的程式碼區塊、HTML `<code>`／`<pre>` 不保護，`--fix` 會改到裡面的字，要保護就改用圍欄。圍欄判斷寧可多保護：縮排程式碼裡單獨出現一行三個反引號時，之後到下一個關閉行都不檢查。
- **只認 `「」`、`『』`**：彎引號 `“”`、`‘’` 與半形引號裡的字照常檢查與修補。Windows 路徑、電子郵件地址不在反引號裡也照常修補。
- **只在同一行內比對**：規則詞被換行拆開、中間夾零寬字元或空白就抓不到；也不檢查簡體字殘留（例如未經 OpenCC 的 `服务器`），全形括號規則不涵蓋半形括號寫法。
- **`參考檔案` 規則來自上游**：在文件語境改成「參考文件」，但「請參考檔案 config.json」這類真的指檔案的句子也會被改，`--fix` 前請看 `[ALERT]`。
- **目錄只收小寫 `.md`**：Linux 上 `.MD`、`.markdown` 不會被收集；目錄裡任何一個 `.md` 是空檔就整次以結束碼 2 中止。
- **只認 LF 與 CRLF**：舊式只有 CR 的換行檔，引號保護的段落邊界判斷不到，`--check` 與 `--fix` 結果可能不一致。
- **模組 API 的 `apply_zhtw_vocab` 預設 `skip_code=False`**（全文替換，相容舊行為）；檢查文件請照上面範例明確傳 `skip_code=True`。`--fix` 以暫存檔替換原檔，硬連結會斷開；符號連結會改寫其指向的檔。
- **不執行 OpenCC**：只做內建詞庫替換。`--verify-idempotent` 驗的是內建規則與重疊樣本，不是 OpenCC 端到端測試，也不是對任意文字的形式證明。
- **`--fix` 只修用詞**，不保證文件事實正確。

後續要補的項目見 [ROADMAP.md](ROADMAP.md)。

## 文件

- [上游簡體開源專案之零漂移繁體中文 Fork 自動同步規範](docs/zhtw-fork-idempotent-sync-spec.md)：對照上游的同步腳本逐段說明「從上游重建分支、OpenCC 轉換、詞庫修補」的做法，以及本工具的實際契約。

## 開發

```bash
# 不要設 PYTHONUTF8，CI 刻意用系統預設編碼
python -m unittest discover -s tests
python tools/verify_zhtw_vocab.py --verify-idempotent --check README.md AGENTS.md ROADMAP.md REVIEW.md docs
```

規範見 [AGENTS.md](AGENTS.md)。

## 致謝與來源

- 詞庫修補加冪等檢查的做法，以及二次退化詞與語境詞的第一批規則，借鑑 Will 保哥 [doggy8088/answer-me-with-html](https://github.com/doggy8088/answer-me-with-html)（`zh-tw` 分支）的 `scripts/zh-tw/vocab.mjs` 與 `check.mjs`。本 repo 是以 Python 重新實作並自行延伸規則，不是上游詞庫的完整複製。
- 本 repo 於 2026-11-19 自 [SanHsien/ai-inbox](https://github.com/SanHsien/ai-inbox) 抽出，保留原檔的 git 歷史。當初的分析筆記與任務紀錄留在 ai-inbox：[保哥繁中 Fork 筆記](https://github.com/SanHsien/ai-inbox/blob/main/notes/2026-11-07-willh-doggy8088-answer-me-with-html-zhtw-fork.md)。

## 授權

[MIT](LICENSE)
