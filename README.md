# cc-token-optimizer-skill

Claude Code skill，用來審查並主動壓縮你電腦上所有 Claude 記憶檔案，縮小每次 session 載入的 context 大小，讓對話更快、更省 token。

---

## ⚠️ 執行前請務必閱讀：這個 skill 會對你的環境做什麼

**直接改動你的檔案，而不只是建議。** 以下逐項說明每個步驟會造成的真實變化：

### 步驟 1 — `.claudeignore`
- **沒有**這個檔案時：在當前目錄**新建** `.claudeignore`，寫入預設忽略規則（node_modules、*.log、*.pyc、*.png 等二進位/產物檔）
- **已存在**時：補上缺少的忽略模式（不刪除你現有的設定）
- **<mark>影響：Claude Code 往後在這個專案讀取檔案時，會跳過上述類型的檔案</mark>**

### 步驟 2 — `~/.claude/settings.json`
**直接修改全域設定檔**，確保以下四個欄位存在且正確：

```json
{
  "model": "opusplan",
  "effortLevel": "high",
  "env": {
    "CLAUDE_CODE_SUBAGENT_MODEL": "haiku",
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "60"
  }
}
```

| 欄位 | 影響 |
|------|------|
| `model: opusplan` | **<mark>主對話使用 Opus 模型 + 思考模式</mark>** |
| `effortLevel: high` | **<mark>自適應推理強度設為 high：Claude 根據任務難度動態分配思考量，複雜任務思考更深，簡單任務仍快速回應（並非每次都用最多思考 token）</mark>** |
| `CLAUDE_CODE_SUBAGENT_MODEL: haiku` | **<mark>所有 subagent（子任務）改用 Haiku 節省 token</mark>** |
| `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE: 60` | **<mark>context 用到 60% 時自動 compact，而非預設的 80–95%（注意：此設定有已知 bug，不保證每次都會生效）</mark>** |

> 修改時只補缺少的欄位，**不會刪除你原有的其他設定**（如 hooks、statusLine 等）。

### 步驟 3 — 全域記憶壓縮（影響最大，請特別注意）

掃描整台電腦所有 Claude 記憶檔案，對每個檔案 **spawn 一個 agent 進行 AI 壓縮，並寫回原路徑**。

**掃描範圍**：
- `~/.claude/CLAUDE.md`（全域指令）
- 所有專案的 `CLAUDE.md`（深度 6 層，排除 node_modules 和 worktrees）
- `~/.claude/projects/*/memory/MEMORY.md`（各專案記憶索引）
- `~/.claude/projects/*/memory/*.md`（各專案個別記憶條目）

**排除**：`~/.claude/skills/`、`node_modules/`、`.claude/worktrees/`、`file-history/`、`backups/`

**壓縮原則**（AI agent 遵循）：
- ✅ 保留所有語意事實（規則、路徑、指令、人名、日期、URL、環境變數名）
- ✅ 保留 YAML frontmatter 與 `Why:` / `How to apply:` 欄位
- ❌ 移除重複論述、冗長範例（同一觀念留一個即可）、填充詞
- 改寫：長敘述 → 條列；冗長前言 → 直接切入

**安全機制**：壓縮任何檔案前，**強制先整批備份**至 `~/.claude/backups/memory-compact-<timestamp>/`。
萬一壓縮結果不滿意，一行指令還原：

```bash
cp -r ~/.claude/backups/memory-compact-<timestamp>/* /
```

> 若掃描到超過 30 個檔案，會先列出全部路徑詢問你確認，再開始壓縮。

- **<mark>影響：所有 Claude 記憶檔案內容將被 AI 改寫並縮短，原檔會被覆蓋；備份自動建立於 `~/.claude/backups/`，可隨時還原</mark>**

### 步驟 4 — MCP 伺服器審查
- **只讀取、不修改任何設定**
- 列出目前連接的 MCP，標注哪些有 CLI 替代方案（如 GitHub MCP → `gh`）
- 建議你手動停用，但不會自動停用
- **<mark>影響：不改動任何檔案，僅提供建議供你自行決定是否停用</mark>**

### 步驟 5 — Context 使用量
- **只讀取、不修改任何設定**
- 告知目前 session context 使用比例，提醒何時應執行 `/compact`
- **<mark>影響：不改動任何檔案，僅回報使用量並建議是否需要手動 /compact</mark>**

---

## 總結：哪些東西會被動到

| 會被改動 | 說明 |
|---------|------|
| 當前目錄的 `.claudeignore` | 新增或補充忽略規則 |
| `~/.claude/settings.json` | 補上缺少的 4 個欄位 |
| 所有 Claude 記憶檔（CLAUDE.md、memory/*.md） | AI 壓縮後寫回原路徑（壓縮前自動備份） |

| 不會被改動 | 說明 |
|-----------|------|
| MCP 設定 | 只報告，不動 |
| 你的程式碼 | 完全不觸碰 |
| `~/.zshrc` 或其他 shell 設定 | 完全不觸碰 |
| Skills 目錄 | 排除在掃描範圍外 |

---

## 安裝

### 前置需求

- [Claude Code](https://claude.ai/code) 已安裝並登入

### 安裝步驟

1. 建立 skill 目錄：

```bash
mkdir -p ~/.claude/skills/token-optimizer
```

2. 下載 skill 檔案：

```bash
curl -o ~/.claude/skills/token-optimizer/skill.md \
  https://raw.githubusercontent.com/dswf65411-new/cc-token-optimizer-skill/main/skill.md
```

或 clone 後 symlink（之後 `git pull` 即可更新）：

```bash
git clone https://github.com/dswf65411-new/cc-token-optimizer-skill.git ~/cc-token-optimizer-skill
mkdir -p ~/.claude/skills/token-optimizer
ln -s ~/cc-token-optimizer-skill/skill.md ~/.claude/skills/token-optimizer/skill.md
```

3. 開新的 Claude Code session，確認 skill 已載入：

```
/skills
```

清單中出現 `token-optimizer` 即安裝成功。

---

## 使用方式

在任何 Claude Code session 中輸入：

```
/token-optimizer
```

或用自然語言觸發：

- 「幫我優化 token 設定」
- 「檢查 Claude Code 設定」
- 「我 token 消耗太快」
- 「幫我做 token 健檢」

Claude 會先建立 5 個任務的清單，再依序執行，每步完成後更新進度。

---

## 適合何時執行

- 剛設定好 Claude Code 環境時（確保設定正確）
- Session context 消耗特別快時
- 記憶檔案累積很多、對話開始變慢時
- 定期維護（例如每個月一次）

---

## 檔案結構

```
cc-token-optimizer-skill/
├── skill.md      # skill 定義檔，Claude Code 載入此檔
└── README.md     # 本說明文件
```

## License

MIT
