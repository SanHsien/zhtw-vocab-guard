# 路線圖

> 本 repo 後續工作的單一真相源。做完一項，就在同一個 commit 刪掉那一列；新工作只加在這裡。
> 現況段是 2026-11-19 查的快照，開工前先重查。

## 現況

台灣繁體中文技術詞彙檢查器（一支工具、33 條規則、39 個測試，只用 Python 標準函式庫），2026-11-19 自 ai-inbox 抽出並保留原檔歷史，MIT 授權。抽出當下只完成本機準備：本機測試與自我檢查通過；GitHub repo 尚未建立，CI、CodeQL、Dependabot 的設定檔已放好但還沒有實際跑過。沒有套件 manifest，依賴新鮮度與歷史壓縮以 `.github/dependabot.yml` 的 waiver 註明理由。

## 近期：可以直接動手

| 項目 | 為什麼 | 驗收方式 | 建議執行者 | 證據 |
| --- | --- | --- | --- | --- |
| 發布到 GitHub 並確認第一次 CI | repo 目前只在本機 | 建立公開 repo `SanHsien/zhtw-vocab-guard` 並推送；`CI`、`CodeQL` 第一次執行皆 success；改寫本檔現況段 | 維護者／整合者 | 本檔現況段 |

## 中期：等條件成立再做

| 項目 | 觸發條件 | 驗收方式 | 建議執行者 | 證據 |
| --- | --- | --- | --- | --- |
| 未加引號的專名保護、`支持→支援` 技術受詞白名單 | 實際出現專名被誤改（例如產品名含 `數據庫`），或需要擋 `支持 UTF-8` 這類用詞 | 先寫出誤改實例的回歸測試再改 `tools/verify_zhtw_vocab.py`；`支持` 只在白名單受詞前替換，`支持開源社群` 等反例不變；`--verify-idempotent --check README.md AGENTS.md ROADMAP.md docs` 通過，並在 ai-inbox 以 `--check notes docs templates README.md` 實跑無新增誤報 | Codex | [ai-inbox 的 speak-human-tw 評估第 6 節](https://github.com/SanHsien/ai-inbox/blob/main/docs/speak-human-tw-assessment.md)；引號、URL 保護與五條候選規則已於 2026-11-15 完成 |

## 刻意不做

| 不做什麼 | 理由 | 什麼情況下重新考慮 |
| --- | --- | --- |
| 內建 OpenCC 或做語義校對 | 定位是固定字串詞庫，只用標準函式庫；OpenCC 轉換由上游流程負責 | 要改變決定，先改這一列再動手 |
| 完整 CommonMark 解析 | 標準函式庫沒有解析器；目前的有限切分已涵蓋圍欄、行內程式碼、連結目的地與引號 | 出現有限切分造成的實際誤改，且無法以回歸測試加小幅修正解決 |
| 把「數位」整詞換成「數字」 | 「數位」是台灣 digital 的正確用詞，整詞替換會大量誤改 | 要改變決定，先改這一列再動手 |
| 不設分支保護與必要檢查 | 單人維護、直推 main 是艦隊慣例；CI 與 CodeQL 紅燈由艦隊稽核與每日巡檢處理 | 有第二位維護者，或開始收外部 PR |

## 維護方式

依 `AGENTS.md` 的開發流程：先寫會紅的測試、修改、測試轉綠、冪等與自我檢查、同步 README 與本檔，再提交。Dependabot 只開 Actions 的 PR；合併前確認 SHA 對應宣稱的版本。
