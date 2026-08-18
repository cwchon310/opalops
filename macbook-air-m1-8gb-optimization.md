# MacBook Air M1 8GB 系統優化完全指南：從「用得耐又慢又滿」到「清爽順手」

> 公開技術文 · 2026-08-19 · macOS 27 · MacBook Air M1 8GB

## 前言

8GB 記憶體嘅 M1 唔代表唔可以用，代表你要**用得聰明**。呢篇綜合 30 幾項實際驗證過嘅優化，分四大部分：

1. **磁碟空間急救**（由幾 GB 垃圾 → 幾 MB）
2. **記憶體與電源管理**（8GB 用得盡又唔卡）
3. **開機與後台瘦身**（邊啲服務唔使留）
4. **終端開發環境加速**（sqlite、快取、shell）

每項都有命令、有預期結果、有注意事項。照抄即用。

---

## Part 1：磁碟空間急救（先攞返幾十 GB 出嚟）

### 1.1 先睇邊度食晒空間

```bash
# 全盤空間
df -h /

# 最肥嘅資料夾 Top 10（掃 /Users 下）
du -sh ~/Library/* 2>/dev/null | sort -rh | head -20
du -sh ~/Library/Caches/* 2>/dev/null | sort -rh | head -15
```

### 1.2 三大「肥佬」：Caches / Logs / 舊快取

**Caches（可以放心清）**：清除後 app 會重新生成，只係第一次開慢少少。

```bash
# 睇下幾肥
du -sh ~/Library/Caches

# 清理前建議先退 app（避免清緊嘅檔被寫入）
# 直接清（只清 Cache，唔影響資料）
rm -rf ~/Library/Caches/*
```

> ⚠️ 唔好刪 `~/Library/Application Support/`，嗰啲係 app 真資料。
> ⚠️ 如果想安全啲，用 `du -sh` 逐個 app 睇，只清最肥嗰幾個。

**Logs**：

```bash
# 系統 log 一般自動循環，但開發工具嘅 log 會無限積
du -sh ~/Library/Logs
# 留最近 7 日
find ~/Library/Logs -type f -mtime +7 -delete 2>/dev/null
```

**開發工具快取（最肥）**：

```bash
# Xcode / Swift 相關
du -sh ~/Library/Developer/Xcode/DerivedData
rm -rf ~/Library/Developer/Xcode/DerivedData/*

# npm / pnpm / yarn
du -sh ~/.npm ~/.pnpm-store ~/.cache 2>/dev/null
npm cache clean --force

# Homebrew
du -sh ~/Library/Caches/Homebrew
brew cleanup -s --prune=all
```

### 1.3 SQLite 資料庫壓縮（神級見效）

開發工具用 SQLite 儲存歷史，用耐咗會「刪咗數據但檔案冇縮」——因為 freed pages 未回收。呢個係最容易被忽略嘅空間黑洞。

```bash
# 例：opencode 資料庫（用過都會脹到幾百 MB）
du -sh ~/.local/share/opencode/opencode.db

# 檢查有幾多「廢頁」
sqlite3 ~/.local/share/opencode/opencode.db \
  "PRAGMA page_count; PRAGMA freelist_count;"

# 如果 freelist 好大 → 壓縮
sqlite3 ~/.local/share/opencode/opencode.db "VACUUM;"
# 試過實際案例：966MB → 1.8MB 🎉
```

**原理**：SQLite 刪除數據後，空白頁會留喺檔入面，VACUUM 會重新整理，將檔壓到真數據大細。

> ⚠️ VACUUM 期間唔好同時開其他 session 寫入，會 lock。
> ⚠️ 壓縮前最好 `sqlite3 db ".backup db.bak"` 做備份。

### 1.4 終端巨檔快取

```bash
# 例：opencode 會將巨型 tool 輸出寫去 tool-output/
du -sh ~/.local/share/opencode/tool-output/
# 搵超肥大檔（>10MB）
find ~/.local/share/opencode/tool-output -size +10M -exec ls -lh {} \;
# 確認係舊 session 殘餘後刪除
rm -f ~/.local/share/opencode/tool-output/tool_<ID>
```

---

## Part 2：記憶體與電源管理（8GB 用到盡）

### 2.1 睇現時記憶體壓力

```bash
# 記憶體壓力（0-100，綠=OK 黃=緊張 紅=爆）
memory_pressure -Q
# 或者用 Activity Monitor 睇 Memory Pressure 圖
```

### 2.2 關閉無謂嘅「記憶體小偷」

```bash
# 列出最食記憶體嘅 app
ps aux -m | head -20
```

**常見 8GB 殺手**：
- 太多瀏覽器分頁 → 用分頁休眠擴充（如 Chrome 內建 Memory Saver）
- Photoshop / 剪片軟件 → 用細檔工作，唔好一次開幾十個
- 太多背景 app → 見 Part 3

### 2.3 避免 swap 過度

8GB 機 swap 用太多會「整個機卡死」。降低 swap 需求嘅方法：

```bash
# 睇 swap 用量
sysctl vm.swapusage
```

如果 swap 長期 >2GB，代表 app 太多。減少同時開嘅 heavy app，比任何優化都有效。

### 2.4 電源／電池保養

```bash
# 睇電池健康
system_profiler SPPowerDataType | grep -A5 "Health Information"
```

**保養貼士**：
- 唔好長期充到 100% 插住電（會蝕電池）
- macOS 有「最佳化電池充電」，保留原廠設定
- 用返原廠充電器，特別係 M1 充電協議

---

## Part 3：開機與後台瘦身（少啲嘢喺背景跑）

### 3.1 睇有咩開機自動啟動

```bash
# 用戶層 LaunchAgents（最常見）
ls -la ~/Library/LaunchAgents/

# 系統層
ls -la /Library/LaunchAgents/ /Library/LaunchDaemons/

# 登入項目（GUI app）
# 系統設定 → 一般 → 登入項目
```

### 3.2 停用／移除唔需要嘅服務

每部 Mac 都可能殘留舊工具嘅 `LaunchAgent`（例如舊備份 script、同步工具）。檢查每一項係咩，唔用就停。

```bash
# 停用（唔刪檔，可以隨時還原）
launchctl unload ~/Library/LaunchAgents/com.example.oldservice.plist

# 或者直接搬走（最徹底，又留後路）
mkdir -p ~/Desktop/launchd-backup
mv ~/Library/LaunchAgents/com.example.oldservice.plist ~/Desktop/launchd-backup/
```

> 💡 原則：**唔係刪除，係搬去備份資料夾**。發現有嘢壞咗可以即刻還原。

### 3.3 檢查 network／VPN／proxy 殘留

有時裝過 VPN / 代理工具，會殘留系統層網路設定拖慢上網：

```bash
# 睇 network 服務
networksetup -listallnetworkservices

# 睇有冇殘留 proxy
scutil --proxy
```

### 3.4 檢查 DNS 被污染／間歇性失敗

呢個係「間歇性收唔到訊息」嘅常見元兇。DNS 偶爾查唔到某個域名，但查其他又正常。

```bash
# 反覆 ping 睇有冇間歇失敗
ping -c 20 api.example.com

# 睇實際解析結果
dig api.example.com +short
nslookup api.example.com

# 如果 DNS 唔穩定，可以直接釘 IP 入 hosts
echo "1.2.3.4 api.example.com" | sudo tee -a /etc/hosts
# 記住先備份 hosts
sudo cp /etc/hosts /etc/hosts.bak
```

> 💡 真實案例：微信橋接收唔到訊息，最後查到係 DNS 對某個域名間歇性 `no such host`。釘 IP 入 `/etc/hosts` 後即刻穩定。

---

## Part 4：終端開發環境加速

### 4.1 終端 shell 提速

```bash
# 睇 shell 啟動時間
time zsh -i -c exit

# 常見拖慢原因：
# 1. 太多 plugin（oh-my-zsh 用 nvm/nvm/brew 每次 init）
# 2. PATH 太長（Homebrew + pyenv + nvm + ...）
# 3. 每次開 shell 都跑大型 script
```

**建議**：
- 只載入你用嘅 plugin
- 用 `zsh` 原生補全而唔係重型框架
- 將慢嘅 tool init 延遲到第一次用先 load（lazy load）

### 4.2 Homebrew 清理

```bash
# 移除已不再需要嘅依賴
brew autoremove

# 清理舊版本 cache
brew cleanup -s

# 睇邊個套件最肥
brew list --formula | xargs -I{} brew info {} --json=v2 | \
  python3 -c "import json,sys; [print(round(d.get('installed',[{}])[0].get('size',0)/1e6,1), d['name']) for d in json.load(sys.stdin)['formulae']]" | sort -rn | head
```

### 4.3 開發工具 session 精簡

長期用 CLI agent（例如 opencode / Claude Code），session 會越積越長越慢。設自動重置：

```toml
# 例：cc-connect 設定，閒置 30 分鐘自動重置 session
[projects.agent.options]
reset_on_idle_mins = 30
```

好處：context 保持精簡，回應快，token 慳。

### 4.4 文件系統優化（APFS 瘦身）

```bash
# 睇「本地快照」有冇食空間（Time Machine 喺度時常見）
tmutil listlocalsnapshots /

# 空間唔夠時可以刪舊快照（會失去該時間點嘅備份）
# 例：刪全部舊 snapshot
tmutil deletelocalsnapshots $(date +%Y-%m-%d)
```

---

## 總結：一個「每週維護」清單

```bash
#!/bin/bash
# weekly-maintenance.sh — 每週行一次

echo "== 1. 清 Caches =="
du -sh ~/Library/Caches
rm -rf ~/Library/Caches/*

echo "== 2. 清 Homebrew =="
brew cleanup -s --prune=all
brew autoremove

echo "== 3. 清 npm 快取 =="
npm cache clean --force 2>/dev/null

echo "== 4. 壓縮 SQLite =="
# 按你實際用嘅 db 逐個 VACUUM

echo "== 5. 睇空間 =="
df -h /
```

儲存做 `weekly-maintenance.sh`，加執行權限：

```bash
chmod +x weekly-maintenance.sh
./weekly-maintenance.sh
```

---

## 常見問題 FAQ

**Q：刪 Cache 會唔會整壞 app？**
A：唔會。Cache 係「重生成快取」，刪咗最多第一次開慢少少。

**Q：VACUUM 有冇危險？**
A：有但可控。先 `sqlite3 db ".backup db.bak"` 做備份，VACUUM 期間唔好同時寫入。

**Q：8GB 夠唔夠日常用？**
A：夠。關鍵係：一次做一件事、後台慳住跑、分頁唔好開爆。M1 嘅統一記憶體 + 快速 swap 已經好能打。

**Q：點知邊個 app 最食記憶體？**
A：`ps aux -m | head -20` 或者 Activity Monitor 排 Memory。

---

## 總結

| 項目 | 做法 | 預期效果 |
|------|------|----------|
| Caches | `rm -rf ~/Library/Caches/*` | 釋放數 GB |
| SQLite VACUUM | `sqlite3 db "VACUUM;"` | 大檔縮至真數據大細 |
| LaunchAgents | 搬走唔用嘅 plist | 開機更快、後台更靜 |
| Homebrew | `brew cleanup -s` | 釋放舊版本 |
| Session 重置 | `reset_on_idle_mins=30` | 開發工具唔會越用越慢 |

**核心心法：唔係「刪」，係「搬」。** 所有優化都留後路（備份檔、backup 資料夾），出事先還原到，先敢放心優化。