English · [繁體中文](README.md)

# zhtw-vocab-guard

A technical vocabulary checker for Traditional Chinese as used in Taiwan. It scans Markdown files for mainland-China technical terms, and for the degraded terms that OpenCC leaves behind when it converts Simplified Chinese to Traditional Chinese. It can check only, or fix files in place. It is a single Python file that uses only the standard library and makes no network calls.

> This is a **fixed-string vocabulary list**, not a semantic proofreader. A pass only means "none of the 34 built-in rules matched"; it does not mean every word is right. For what it cannot catch, see "Known limitations" below.

## Where it sits in a documentation workflow

```
 write docs / OpenCC conversion ──► --check (CI or pre-commit) ──► [ALERT] found ──► --fix in place ──► --check again
                                        │                                               │
                                        ▼                                               ▼
                                proceed only on exit code 0            replaces until a fixed point; rerunning changes nothing
```

## Installation

Requires Python 3.11 or later. No packages to install.

```bash
git clone https://github.com/SanHsien/zhtw-vocab-guard.git
cd zhtw-vocab-guard
python tools/verify_zhtw_vocab.py --help
```

To use it in another repo, call the file by absolute path, or copy `tools/verify_zhtw_vocab.py` over (single file, no dependencies).

## Usage

| Command | Purpose |
|---------|---------|
| `python tools/verify_zhtw_vocab.py --check <path>...` | Check only. Directories are searched recursively for `.md` files; files named explicitly are processed as given. Each hit prints one `[ALERT] file:line` line |
| `python tools/verify_zhtw_vocab.py --fix <path>...` | Fix in place, keeping each file's line endings. It first checks every input, computes every rewrite, and confirms that every file to be changed is writable; only then does it write. Each file is replaced atomically through a temporary file and `os.replace`, so a failure mid-write does not leave a truncated file. There is no transactional rollback across files: if replacing file N fails, the files already replaced keep their fixed content |
| `python tools/verify_zhtw_vocab.py --verify-idempotent` | Verify that the built-in rules are idempotent: fixing already-fixed text a second time changes nothing |

The three options can be combined. A typical CI invocation:

```bash
python tools/verify_zhtw_vocab.py --verify-idempotent --check README.md docs
```

Exit codes:

| Exit code | Meaning |
|-----------|---------|
| 0 | Every requested operation passed |
| 1 | Vocabulary problems were found, or an idempotency sample failed |
| 2 | Argument, input, or I/O error: no operation given, path does not exist, no files to check, a file is empty, a file is not UTF-8 (the error message names the file), or a `--fix` target is not writable |

`[REVIEW]` only asks for a manual look (when a bare URL contains a rule term, the original text is kept and flagged) and does not affect the exit code. Only `[ALERT]` makes the check fail.

It can also be used as a module:

```python
from verify_zhtw_vocab import apply_zhtw_vocab, scan_degraded_vocab

fixed = apply_zhtw_vocab(text, skip_code=True)   # returns the fixed text
hits = scan_degraded_vocab(text)                 # returns the list of hits (term, replacement, reason, line)
```

## Rule coverage

There are 34 rules in two groups, defined as two constants at the top of `tools/verify_zhtw_vocab.py`:

- **`GLOBAL_VOCAB_RULES` (21 rules)**: high-confidence technical compounds and action words. Examples: `服務器`, `軟件`, `硬件`, `視頻`, `數據庫`, `環境變量`, `開源代碼`, `源代碼`, `命令行`, `正則表達式`, `默認值`, `中間件`, `重定向`, `粘貼`, `摁下`.
- **`DEGRADED_AND_CONTEXT_RULES` (13 rules)**: terms degraded by a second OpenCC conversion (`cn` → `tw2`), and context terms that only apply inside specific phrases. Examples: `連線埠`, `隻影響`, `型別別`, `瞭什麼`, `參考檔案`, `關鍵數位`.

What is left untouched (`skip_code=True`, the CLI default):

- Markdown fenced code blocks, and backtick runs of equal length on the same line. To cite a wrong term as a counter-example in a document, put it in backticks. A fence may be preceded by a BOM, by space or tab indentation (fences inside lists), by the blockquote marker `>`, and by list markers.
- Quoted speech inside paired Chinese quotation marks (`「」`, `『』`): at most 200 characters, not spanning a blank-line paragraph break or a table cell. Mismatched quotation marks are not protected.
- URLs and link destinations: bare URLs, autolinks, and the destinations of inline and reference-style links. Link text and link titles are checked as usual.

A few rules carry exceptions on the neighbouring characters so that they do not cut into other words. For example, `電視頻道`, `巡視頻率`, `型別別名`, `資源代碼`, `命令行為`, and `明瞭什麼` are not hits; `開源代碼` becomes `開源程式碼` and is not cut into `開原始碼`.

## Known limitations

- **No proper-noun protection**: an unquoted product name that contains a rule term is changed like any other text.
- **No semantic judgement**: it does not look at part of speech, headings, or punctuation, and does not judge whether a sentence reads well.
- **`支持` is not automatically replaced with `支援`**: both are correct in Taiwan, and choosing between them depends on the object of the verb. This is not covered yet.
- **`數位` is covered by two phrases only**: in Taiwan, `數位` is the correct word for "digital", so only two phrases where "number" was wrongly converted (the `同樣的…` and `關鍵…` phrases) are covered. Wrong conversions elsewhere are not caught, and digital compounds outside the exception character set may be changed by mistake.
- **Neighbouring-character exceptions are character-set matches**: rare combinations can still be wrongly excluded (for example `接收` immediately followed by the mainland spelling of "video").
- **Markdown parsing is limited**: it is not full CommonMark. Inline code spanning multiple lines is not supported, and link destinations are only recognised on a single line. Four-space indented code blocks and HTML `<code>` / `<pre>` are not protected, so `--fix` changes text inside them; use fences to protect such text. Fence detection errs towards protecting more: when a line of three backticks appears alone inside indented code, nothing is checked from there to the next closing line.
- **Only `「」` and `『』` are recognised**: text inside curly quotes `“”`, `‘’` and ASCII quotes is checked and fixed as usual. Windows paths and email addresses outside backticks are also fixed as usual.
- **Matching happens within a single line**: a rule term split by a line break, or with a zero-width character or whitespace in the middle, is not caught. Leftover Simplified characters (for example `服务器` that never went through OpenCC) are not checked, and the full-width bracket rule does not cover the half-width form.
- **The `參考檔案` rule comes from upstream**: in a documentation context it becomes `參考文件`, but a sentence that really means a file, such as `請參考檔案 config.json`, is changed too. Read the `[ALERT]` lines before running `--fix`.
- **Directories only collect lowercase `.md`**: on Linux, `.MD` and `.markdown` are not collected. If any `.md` in a directory is empty, the whole run stops with exit code 2.
- **Only LF and CRLF are recognised**: in files with old CR-only line endings, paragraph boundaries for quotation-mark protection cannot be determined, so `--check` and `--fix` may disagree.
- **The module API's `apply_zhtw_vocab` defaults to `skip_code=False`** (replace across the whole text, for backward compatibility). When checking documents, pass `skip_code=True` explicitly as in the example above. `--fix` replaces the original through a temporary file, which breaks hard links; for a symbolic link, the file it points to is rewritten.
- **It does not run OpenCC**: it only applies the built-in vocabulary replacements. `--verify-idempotent` verifies the built-in rules and overlapping samples; it is not an end-to-end OpenCC test and not a formal proof for arbitrary text.
- **`--fix` only fixes wording**; it does not guarantee that a document's facts are correct.

For what is planned next, see [ROADMAP.md](ROADMAP.md).

## Documentation

- [Zero-drift Traditional Chinese fork sync spec for upstream Simplified Chinese open-source projects](docs/zhtw-fork-idempotent-sync-spec.md) (in Traditional Chinese): walks through the upstream sync script section by section ("rebuild the branch from upstream, convert with OpenCC, apply vocabulary fixes"), and states this tool's actual contract.

## Development

```bash
# Do not set PYTHONUTF8; CI deliberately uses the system default encoding
python -m unittest discover -s tests
python tools/verify_zhtw_vocab.py --verify-idempotent --check README.md AGENTS.md ROADMAP.md REVIEW.md docs
```

The rules for contributors are in [AGENTS.md](AGENTS.md).

## Acknowledgements and origin

- The approach of vocabulary fixes plus an idempotency check, and the first batch of degraded-term and context-term rules, draw on `scripts/zh-tw/vocab.mjs` and `check.mjs` in Will Huang's [doggy8088/answer-me-with-html](https://github.com/doggy8088/answer-me-with-html) (`zh-tw` branch). This repo is a Python reimplementation with its own extended rules, not a complete copy of the upstream vocabulary.
- This repo was extracted from [SanHsien/ai-inbox](https://github.com/SanHsien/ai-inbox) on 2026-11-19, keeping the git history of the original files. The original analysis note and task records stay in ai-inbox: [note on Will Huang's Traditional Chinese fork](https://github.com/SanHsien/ai-inbox/blob/main/notes/2026-11-07-willh-doggy8088-answer-me-with-html-zhtw-fork.md).

## License

[MIT](LICENSE)
