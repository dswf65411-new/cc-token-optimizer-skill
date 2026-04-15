---
name: token-optimizer
description: 審查並優化 Claude Code 的 token 使用設定。當使用者說「幫我優化 token 設定」、「檢查 Claude Code 設定」、「我 token 消耗太快」、「幫我做 token 健檢」、「跑 token-optimizer」時觸發。新專案開始時、或感覺 session 消耗太快時使用。
---

# Token Optimizer

審查當前專案的 Claude Code token 使用設定，逐項檢查並直接修正問題。

## 執行原則

- **開始前先用 `TaskCreate` 建立完整任務清單**，包含以下 6 個任務，再開始執行
- 每完成一項立即呼叫 `TaskUpdate` 將狀態標為 `completed`，不得等到最後才批次更新
- 每一項都告訴使用者檢查結果（✅ 已設定 / ⚠️ 需要修正）
- 可以直接修正的就直接修正，不要只說「建議你去改」
- 需要使用者確認再動的（如刪 CLAUDE.md 內容），先列出來讓他確認
- 全部跑完後輸出一份摘要

## 任務清單（開始前建立）

執行第一步前，先呼叫 `TaskCreate` 建立以下 6 個任務：

1. **檢查 .claudeignore** — 確認專案根目錄是否存在 .claudeignore，缺少則建立
2. **檢查 settings.json** — 確認 model: opusplan、effortLevel: high、env.CLAUDE_CODE_SUBAGENT_MODEL: haiku 設定正確
3. **全域記憶壓縮** — 掃描全機器記憶檔 → 備份 → 平行派工 → 壓縮回報
4. **審查 MCP 伺服器** — 列出可用 CLI 替代的 MCP，建議使用者停用
5. **檢查 Context 使用量** — 確認目前 session context 健康度

---

## 檢查清單

### 1. `.claudeignore`

> 完成後呼叫 `TaskUpdate` 將任務 1 標為 `completed`。

檢查專案根目錄（當前工作目錄）是否存在 `.claudeignore`。

**不存在時**：直接建立，寫入以下內容：

```
# 建置產物
node_modules/
dist/
build/
.next/
.nuxt/
out/
coverage/

# 套件管理
*.lock
package-lock.json
yarn.lock
pnpm-lock.yaml
Cargo.lock
Gemfile.lock
poetry.lock

# 日誌與暫存
*.log
*.tmp
.cache/
.parcel-cache/
tmp/

# 編譯產物
__pycache__/
*.pyc
*.pyo
target/
.gradle/
*.class

# IDE 與系統
.DS_Store
Thumbs.db
.idea/
*.swp

# 資料與媒體（通常不需要 Claude 讀）
*.csv
*.xlsx
*.pdf
*.png
*.jpg
*.jpeg
*.gif
*.mp4
*.zip
*.tar.gz
```

**已存在時**：讀取內容，確認是否缺少上述關鍵項目，缺少就補上。

---

### 2. `~/.claude/settings.json` 全域設定

> 完成後呼叫 `TaskUpdate` 將任務 2 標為 `completed`。

讀取 `~/.claude/settings.json`（不存在就建立）。

確認以下欄位存在且正確：

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

- `model` 不是 `opusplan` → 修改為 `opusplan`
- `effortLevel` 不存在 → 加入 `"effortLevel": "high"`
- `env.CLAUDE_CODE_SUBAGENT_MODEL` 不是 `"haiku"` → 補上或修正
- `env.CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` 不是 `"60"` → 補上或修正（讓 context 用到 60% 時自動 compact，取代預設的 80-95%）
- 檔案不存在 → 建立並寫入上述內容

> 注意：修改時保留使用者原有的其他設定，只補充缺少的欄位，不要覆蓋整個檔案。

---

### 3. 全域記憶壓縮工作流

> 完成後呼叫 `TaskUpdate` 將任務 3 標為 `completed`。

四個子步驟依序執行。

#### 3-1. Discovery（CLI 掃描 → manifest）

用 Bash 工具執行以下指令，把全部要處理的檔案路徑寫到 manifest（路徑含時間戳以免衝突）：

```bash
TS=$(date +%Y%m%d-%H%M%S)
MANIFEST=~/Downloads/claude-memory-manifest-$TS.txt
{
  # Global CLAUDE.md
  [ -f ~/.claude/CLAUDE.md ] && echo ~/.claude/CLAUDE.md

  # Project CLAUDE.md（排除 skills / worktrees / global 自己）
  find ~ -maxdepth 6 -name "CLAUDE.md" \
    -not -path "*/node_modules/*" \
    -not -path "*/.claude/skills/*" \
    -not -path "*/.claude/worktrees/*" \
    -not -path "$HOME/.claude/CLAUDE.md" \
    2>/dev/null

  # Auto-memory：MEMORY.md + 個別記憶檔（排除 file-history / backups）
  find ~/.claude/projects -type f -name "*.md" \
    -not -path "*/file-history/*" \
    -not -path "*/backups/*" \
    2>/dev/null
} | sort -u > "$MANIFEST"

echo "Manifest: $MANIFEST"
wc -l "$MANIFEST"
```

印出總檔案數後繼續。

#### 3-2. 備份（強制，壓縮前執行）

把 manifest 裡的所有檔案整批備份至 `~/.claude/backups/memory-compact-<timestamp>/`，保留原路徑結構：

```bash
BACKUP=~/.claude/backups/memory-compact-$TS
mkdir -p "$BACKUP"
while IFS= read -r f; do
  rel="${f#/}"
  mkdir -p "$BACKUP/$(dirname "$rel")"
  cp "$f" "$BACKUP/$rel"
done < "$MANIFEST"
echo "Backup done: $BACKUP"
```

#### 3-3. TaskCreate

讀取 manifest，對**每一個檔案**建立一個任務（title: `壓縮 <basename>`，description: 完整路徑）。

若 manifest 檔案數 > 30，在 TaskCreate 前列出全部路徑，先問使用者「全部都要壓縮嗎？」（AskUserQuestion），確認後再建任務。

#### 3-4. 平行壓縮（Agent 分波派工）

用 `Agent` tool（`subagent_type="general-purpose"`）每個檔案派一個 agent 壓縮。每波同時並發 **8 個 Agent tool calls**（單一訊息多 tool use），等該波全部回傳後再發下一波。每個任務完成後立即 `TaskUpdate` 標為 `completed`。

**每個 agent prompt 必須包含（self-contained）**：

```
你的任務是壓縮以下 Claude 記憶檔，在不流失任何語意的前提下減少行數：

FILE: <完整路徑>

壓縮原則：
- 保留所有語意事實：規則、偏好、路徑、指令、人名、日期、URL、環境變數名
- 保留 YAML frontmatter（name / description / type 欄位完整保留）
- 保留 "Why:" 和 "How to apply:" 欄位（若存在）
- 移除：重複論述、同一觀念的多個範例（留最代表性的一個）、過場句、填充詞
- 改寫：長敘述 → 條列；冗長前言 → 直接切入重點

依檔名類型採不同策略：
- MEMORY.md：索引項不得刪減，只壓縮每行描述至 ≤ 80 字元
- CLAUDE.md（global / project）：可重組段落、合併重複規則
- <type>_<name>.md（記憶檔）：frontmatter 完整保留，內文積極壓縮

硬性規則：
- 不得刪除任何一個可獨立成立的事實點
- 不確定是否可壓的段落 → 保留原文
- 用 Write tool 寫回同一路徑（覆蓋原檔）

最後必須輸出：
FILE: <path>
BEFORE: <lines> lines / <chars> chars
AFTER:  <lines> lines / <chars> chars
REDUCTION: <pct>%
```

#### 3-5. 收斂報告

全部 agent 回傳後，主 session 彙整輸出：

```
━━━━━━━━━━━━━━━━━━━━━━━━
🗜  記憶壓縮報告
━━━━━━━━━━━━━━━━━━━━━━━━
檔案類型            檔案數   原始行數    壓縮後行數   縮減
Global CLAUDE.md       1      XXX         XXX       XX%
Project CLAUDE.md      N      XXX         XXX       XX%
MEMORY.md              N      XXX         XXX       XX%
個別記憶檔             N      XXX         XXX       XX%
──────────────────────────────────────────────────────
合計                   N      XXX         XXX       XX%

備份：~/.claude/backups/memory-compact-<timestamp>/
失敗檔（若有）：<列表>
還原：cp -r ~/.claude/backups/memory-compact-<timestamp>/* /
━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### 4. MCP 伺服器審查

執行 `/mcp` 或讀取設定確認目前連接的 MCP 伺服器清單。

對每個 MCP 伺服器，判斷是否有對應的 CLI 工具可以替代：

| MCP 伺服器 | 可替代的 CLI |
|-----------|------------|
| GitHub MCP | `gh` |
| AWS MCP | `aws` |
| GCP MCP | `gcloud` |
| Slack MCP | `slack` CLI |
| Linear MCP | `linear` CLI |
| 資料庫查詢類 | 直接用 `psql`、`mysql` 等 |

**對每個可替代的 MCP**：告知使用者「這個 MCP 可以改用 CLI 工具，建議停用以省 token」，但**不要自動停用**，讓使用者決定。

**沒有連接任何 MCP**：✅ 告知不需要處理。

> 完成後呼叫 `TaskUpdate` 將任務 4 標為 `completed`。

---

### 5. 當前 session context 使用狀況

執行 `/context` 查看目前 context 使用量。

- 低於 40%：✅ 健康
- 40–60%：⚠️ 提醒使用者「快到建議 /compact 的時機了（建議在 60% 前執行）」
- 超過 60%：🔴 建議立即執行 `/compact`

> 完成後呼叫 `TaskUpdate` 將任務 5 標為 `completed`。

---

## 輸出摘要格式

所有檢查跑完後，輸出以下格式的摘要：

```
━━━━━━━━━━━━━━━━━━━━━━━━
🔍 Token Optimizer 健檢報告
━━━━━━━━━━━━━━━━━━━━━━━━

✅ .claudeignore        已設定／已建立
✅ settings.json        model: opusplan, effort: high
⚠️  CLAUDE.md           超過 200 行，已列出建議（待你確認）
✅ MCP 伺服器           無需調整
✅ Context 使用量       目前 23%，健康

需要你確認的項目：[列出項目]
已自動修正的項目：[列出項目]
━━━━━━━━━━━━━━━━━━━━━━━━
```
