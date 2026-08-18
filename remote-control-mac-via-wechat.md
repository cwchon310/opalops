# 用手機遙距控制你部電腦：微信 × AI 助手橋接完全教學

> 公開技術文 · 2026-08-19 · 適用 macOS / Linux，Windows 原理相通

## 前言

想像一個畫面：你喺街，人哋喺微信傳個檔案俾你，話「幫我睇下呢個」，你部電腦就自己解壓、分析、出報告，傳返俾你——全程你淨係喺手機打幾句字。

呢篇教嘅正正就係呢件事：**用微信做入口，遙距指揮你部電腦上嘅 AI 助手**（opencode / Claude Code 等命令行 AI agent）做嘢。

## 點解要咁做？

| 方式 | 缺點 |
|------|------|
| TeamViewer / 遠端桌面 | 食電、慢、要手機 app、畫質差 |
| SSH from phone | 冇好嘅手機 SSH 客戶端，打字好痛苦 |
| 直接用網頁 AI | 睇唔到你電腦嘅檔案 |
| **微信 → AI agent** | 手機已經有微信、打字快、AI 幫你執行，唔使「操控螢幕」 |

核心思維轉變：**你唔使遙距「操控」部電腦，你只需要遙距「落指令」**，AI 幫你執行。

---

## Part 1：整個架構係點

```
手機微信
   │ 傳訊息（自然語言指令）
   ▼
cc-connect（微信橋接 daemon，長駐你部電腦）
   │ 監聽微信個人號訊息
   ▼
opencode（命令行 AI agent）
   │ 執行：讀檔、寫 code、跑命令、整報告
   ▼
cc-connect 將結果傳返去微信
   ▲
手機收到結果（文字／檔案）
```

三個組件：
1. **微信個人號** —— 你手機同電腦兩邊都登入緊（透過 ilink/第三方橋接）
2. **cc-connect** —— 喺電腦行嘅 daemon，負責「微信 ↔ 電腦」雙向通道
3. **opencode** —— 你嘅 AI 助手本體，負責真正執行任務

> 💡 概念唔限於微信：同一套嘢可以駁去 Telegram、Discord、Slack。原理一樣，只係橋接層唔同。

---

## Part 2：安裝 cc-connect

### 2.1 安裝

```bash
# macOS / Linux 通用（睇你項目官方文件）
# 通常係下載 binary 或者用套件管理器
brew install cc-connect   # 假設 macOS 有 formula（按實際情況）
# 或者直接下載 release binary 放去 /usr/local/bin
```

### 2.2 設定

cc-connect 用 TOML 設定檔，例如 `~/.cc-connect/config.toml`：

```toml
# 專案：一個「工作目錄」一個 project
[projects.agent]
workdir = "/home/user/projects/wechat-agent"   # 你嘅工作目錄

[projects.agent.agent]
command = "opencode"        # 要啟動嘅 AI agent

[projects.agent.platform]
type = "weixin"             # 橋接平台
name = "ilink"              # 第三方微信接入
```

每個設定檔都有詳細註解，照住填就得。

### 2.3 啟動（長駐）

```bash
cc-connect start
# 用 launchd / systemd 設定成開機自動啟動

# macOS：LaunchAgent
# Linux：systemd service
```

設定好之後，**微信就可以開始同你部電腦對話**。

---

## Part 3：第一次對話

### 3.1 基本指令

喺微信傳俾你部電腦：

```
hi
```

如果設定正確，會收到類似「我係你嘅 AI 助手，有咩幫到你」嘅回覆。

### 3.2 常用操作範例

**查檔案／睇狀態：**
```
睇下而家資料夾有咩檔
部機 CPU 溫度幾多？
```

**執行命令：**
```
幫我 check 下 8080 port 係咪開住
寫個 script 定期備份 ~/Documents
```

**出報告：**
```
睇下 /tmp 有咩大檔，列一個 Top 10 清單俾我
```

**檔案處理：**
```
解壓呢個 zip，睇下入面有咩，整理一個摘要俾我
```

---

## Part 4：進階功能

### 4.1 Cron 排程（唔使指令，時間到自動做）

cc-connect 支援「預約任務」，類似 cron 但由電腦主動執行：

```toml
[projects.agent.crons]
"0 9 * * *" = "總結尋日嘅工作，寫成報告放去 ~/reports/"
"*/30 * * * *" = "check 下系統健康，有異常先通知"
```

效果：每日朝早 9 點自動出昨日總結，每 30 分鐘自查健康——**全部唔使你出手**。

### 4.2 檔案傳送

- 電腦 → 手機：AI 執行完任務可以直接將報告／檔案送返微信
- 手機 → 電腦：你傳個檔案俾電腦，AI 收到後可以分析佢

### 4.3 語音（可選）

部分橋接支援將語音訊息轉文字，你喺街直接講句「幫我記住買牛奶」都得。

### 4.4 多 Agent / 多專案

一個 daemon 可以開多個 project，各自獨立工作目錄同 agent，唔會互相干擾。

---

## Part 5：穩定運行貼士

### 5.1 DNS 問題（真實踩坑）

橋接用耐咗可能出現「間歇性收唔到訊息」。常見原因：**DNS 對橋接伺服器域名間歇性查詢失敗**。

```bash
# 反覆測試
ping -c 20 bridge.example.com
nslookup bridge.example.com

# 如果間歇性失敗 → 釘 IP 入 hosts
echo "1.2.3.4 bridge.example.com" | sudo tee -a /etc/hosts
```

> ⚠️ 先備份：`sudo cp /etc/hosts /etc/hosts.bak`

真實案例：一個星期內 98 次 `no such host`，釘 IP 後完全恢復。

### 5.2 Session 越用越慢（自動重置）

長期連住，AI session 會越積越長、回應越嚟越慢。設自動重置：

```toml
[projects.agent.options]
reset_on_idle_mins = 30   # 閒置 30 分鐘自動重置 session
```

### 5.3 斷線自動重連

確保 daemon 有自動重連邏輯（大部分有），並設定成開機自啟，唔好手動開。

### 5.4 安全注意

- **唔好用生產機**：測試環境用先安樂
- **設好防火牆**：電腦只俾必要端口對外
- **唔好講密碼／機密資料入微信**：微信橋接層會經第三方伺服器
- **定期睇 log**：有異常早發現

```bash
cc-connect logs --follow   # 或者睇 ~/.cc-connect/logs/
```

---

## Part 6：常見問題 FAQ

**Q：一定要微信？**
A：唔係。Telegram / Discord / Slack 都得，原理一樣。揀你常用嗰個。

**Q：電腦一定要長開？**
A：係。daemon 要長駐。可以諗用部 Raspberry Pi 做「哨兵機」24 小時開住。

**Q：安唔安全？**
A：取決於你點用。橋接會經第三方伺服器，所以：唔講機密、設防火牆、定期檢查 log。

**Q：AI 做錯嘢點算？**
A：opencode 有 permission 系統，破壞性命令會問准先做（喺微信顯示確認）。亦可以設白名單命令。

**Q：打字好慢，有冇捷徑？**
A：設定常用「快捷指令」（例如 `dd` = 今日總結），或者用 cron 自動觸發。

---

## 總結

| 組件 | 角色 | 一句話 |
|------|------|--------|
| 微信 | 輸入／輸出 | 你喺手機打字，收結果 |
| cc-connect | 橋接 | 微信 ↔ 電腦雙向通道 |
| opencode | 執行者 | AI 真正做嘢 |

**最關鍵嘅概念：唔係遙距操控，係遙距指揮。**

你唔使睇住個螢幕郁滑鼠，你只需要講「做咩」，AI 會話你知「結果」。呢個先係現代遙距工作嘅正確姿勢：**人落指令，AI 執行，結果自動送返你手。**

---

## 進階延伸（想再玩深啲）

- **接 Telegram**：換 platform 設定就切換
- **接 cron**：定時自動任務（日報、監控、備份）
- **多專案**：一個 daemon 管多個工作目錄
- **群組用**：揀支援群組嘅橋接，成隊人共用一個電腦 agent
- **TTS 語音回覆**：部分橋接支援將結果講出嚟，駕駛時都用到

由「遙距睇」升級做「遙距做」，呢個就係橋接 AI 助手嘅最大價值。