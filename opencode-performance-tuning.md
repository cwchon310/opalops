# opencode 全面效能優化：從 1.8GB 快取到更快回應

> 公開技術文 · 2026-08-19 · opencode v1.18.18

## 前言

opencode 用耐咗會發現越嚟越慢。呢篇記錄三項實際見效嘅優化：快取清理、資料庫壓縮、skills 瘦身。

## 1. 清 tool-output 快取（1.8GB → 6MB）

opencode 將超過限制嘅 tool 輸出寫去 `~/.local/share/opencode/tool-output/`。大型 bash 輸出（例如 dump 二進制）可以單檔幾百 MB。

```bash
du -sh ~/.local/share/opencode/tool-output/
# 發現兩個 909MB 巨型檔
find ~/.local/share/opencode/tool-output -size +10M -exec ls -lh {} \;

# 確認係舊 session 嘅殘餘後刪除
rm -f ~/.local/share/opencode/tool-output/tool_<ID>
```

> 注意：只刪舊嘅、唔屬於活躍 session 嘅檔。刪錯會令某啲歷史 tool 輸出回唔到。

## 2. 壓縮 SQLite 資料庫（966MB → 1.8MB）

opencode 用 SQLite（`~/.local/share/opencode/opencode.db`）。長期使用後 file 會脹到幾百 MB，即使實際數據只有幾 MB——因為大量 freed pages 未回收。

```bash
sqlite3 ~/.local/share/opencode/opencode.db "PRAGMA page_count; PRAGMA freelist_count;"
# 若 freelist 好大 → 需要 VACUUM
opencode db "VACUUM;"
```

結果：966MB → 1.8MB，disk I/O 同 backup 都快好多。

## 3. Skills 瘦身（132 → 79）

opencode 每次 session 都會載入**全部 skills 嘅描述**入 context。132 個 skills = 每 turn 都要食呢啲 token，又慢又分散注意力。

做法：

```bash
# 盤點
ls ~/.config/opencode/skills/
# 分類：真正會用 vs 裝咗未用
# 例：33 個 firecrawl skills（要 API key 但從未用）、claude-api（1MB 但用緊 deepseek）...
mkdir -p ~/.config/opencode/backups/skills-prune-<date>
mv ~/.config/opencode/skills/firecrawl* ~/.config/opencode/backups/skills-prune-<date>/
```

**原則**：
- 唔係刪除，係搬去 backup——可以隨時還原。
- 保留同你工作相關嘅、同核心工作流相關嘅。
- 每次 session 載入嘅 context 大減 → 回應更快、技能匹配更準。

## 4. 防再肥大：session 自動重置

如果透過 cc-connect 長期遙距用 opencode，session 會越積越長越慢。設自動重置：

```toml
[projects.agent.options]
reset_on_idle_mins = 30
```

閒置 30 分鐘後 session 自動重置，context 保持精簡。

## 總結對比

| 項目 | 優化前 | 優化後 |
|------|--------|--------|
| tool-output | 1.8GB | 6MB |
| opencode.db | 966MB | 1.8MB |
| skills | 132 | 79 |
| 每次 session context | 肥大 | 精簡 |

## 注意

- VACUUM 期間唔好開住其他 opencode session 一齊寫入，會 lock。
- 刪巨型 tool-output 前確認唔係活躍 session 緊用。