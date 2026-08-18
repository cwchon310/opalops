# macOS 終端命令大全：零基礎日常必備 100+ 命令

> 公開技術文 · 2026-08-19 · macOS 27 / zsh

## 前言

「終端」係 macOS 入面最強大嘅工具，但新手通常只識 `ls`。呢篇由零開始，將日常用得著嘅命令全部整理好，分成九大類：

1. **基礎檔案操作**（睇、開、搬、刪）
2. **系統資訊**（CPU、記憶體、磁碟、電池）
3. **網絡工具**（ping、curl、掃 port）
4. **程式與進程**（開 app、睇進程、殺進程）
5. **磁碟與空間**（搵大檔、清理）
6. **權限與用戶**（sudo、chmod、keychain）
7. **安裝套件**（Homebrew）
8. **自動化**（alias、script、cron/launchd）
9. **救援技巧**（Mac 死機/卡住/裝唔到嘢）

**點用呢篇**：唔使死記，揀你而家想做嘅事，抄命令即刻用。

---

## 入門：終端基本操作

```bash
pwd               # 我喺邊（print working directory）
cd ~/Documents    # 轉去資料夾
cd ..             # 上一層
cd ~              # 回家
ls                # 列出檔案
ls -la            # 詳細列出（含隱藏檔）
clear             # 清畫面
history           # 睇歷史命令
!!                # 重跑上一個命令
```

> 💡 Tab 鍵自動補齊，上/下方向鍵翻歷史命令。

---

## Part 1：基礎檔案操作

```bash
# 睇檔案內容
cat file.txt
less file.txt     # 分頁睇（q 退出）
head -20 file.txt # 睇頭 20 行
tail -20 file.txt # 睇尾 20 行
tail -f file.log  # 實時跟 log

# 開檔案／資料夾
open file.txt               # 用預設 app 開
open .                      # 開當前資料夾（Finder）
open -a "Safari" https://x  # 指定 app 開

# 建立／複製／移動／刪除
mkdir new-folder
cp file.txt backup.txt
mv file.txt new-location/
rm file.txt
rm -r folder/          # 刪資料夾（連內容）
rm -rf folder/         # 強制刪（小心！）

# 搵檔案
find . -name "*.md"
mdfind "keyword"       # Spotlight 搜尋

# 打包／解壓
zip -r archive.zip folder/
unzip archive.zip
tar -czf archive.tar.gz folder/
tar -xzf archive.tar.gz

# 內容搜尋
grep -r "keyword" .
rg "keyword" .         # 更快，ripgrep
```

---

## Part 2：系統資訊

```bash
# 硬件
system_profiler SPHardwareDataType   # CPU/記憶體/型號
sysctl -n machdep.cpu.brand_string   # CPU 型號
sysctl -n hw.memsize                 # 記憶體 byte
sysctl -n hw.ncpu                    # CPU 核數

# 系統版本
sw_vers
uname -a

# 記憶體壓力
memory_pressure -Q
vm_stat

# 磁碟
df -h
diskutil list

# 電池
system_profiler SPPowerDataType | grep -A5 "Health"

# 開機時間
uptime
```

---

## Part 3：網絡工具

```bash
# 基本測試
ping -c 4 example.com
dig example.com +short     # DNS 解析
nslookup example.com

# 下載
curl -O https://example.com/file.zip
curl -o name.zip URL       # 改名下載

# API 測試（好常用！）
curl -s https://api.example.com/v1/data
curl -s -X POST https://api.example.com/v1/item \
  -H "Content-Type: application/json" \
  -d '{"name":"test"}'

# 睇自己 IP
curl -s ifconfig.me        # 公網 IP
ipconfig getifaddr en0     # 內網 IP

# 掃 port
nc -zv example.com 80       # 測某 port 開唔開
# 掃一段
for p in $(seq 1 100); do nc -z -w1 example.com $p && echo "open: $p"; done

# 監聽緊嘅 port
sudo lsof -iTCP -sTCP:LISTEN -P
lsof -i :8080              # 邊個 program 用緊 8080

# Wi-Fi
networksetup -getairportnetwork en0
```

---

## Part 4：程式與進程

```bash
# 睇進程
ps aux | head -20
ps aux -m | head -10        # 最食記憶體 Top 10
top -l 1 | head -20
htop                        # 互動式（brew install htop）

# 殺進程
kill 1234                   # 正常終止 PID 1234
kill -9 1234                # 強制殺
pkill -f "processname"      # 按名字殺

# 開 app
open -a "Calculator"
open -a "Terminal"
brew services list          # 睇 Homebrew 服務

# 搵邊個食緊 CPU 大
ps aux --sort=-%cpu | head -10
```

---

## Part 5：磁碟與空間

```bash
# 全盤
df -h

# 資料夾大細（好常用）
du -sh ~/Library/Caches
du -sh ~/*

# 最肥 Top 10
du -sh ~/Library/* 2>/dev/null | sort -rh | head -10

# 搵超過 1GB 嘅檔案
find ~ -size +1G -exec ls -lh {} \; 2>/dev/null

# 睇 APFS 快照（Time Machine 用）
tmutil listlocalsnapshots /

# 睇系統報表
system_profiler SPStorageDataType
```

---

## Part 6：權限與用戶

```bash
# sudo（管理員權限）
sudo command

# 檔案權限
ls -la                    # 睇權限
chmod +x script.sh        # 加執行權限
chmod 755 file            # 標準執行檔
chmod 600 file            # 得自己讀寫（密鑰檔用呢個！）
chmod 700 folder          # 資料夾，得自己

# 用戶
whoami
id
dscl . -list /Users      # 列出用戶

# 密碼管理（keychain）
security find-internet-password -s github.com
```

---

## Part 7：Homebrew 套件管理

```bash
# 安裝 Homebrew（一次）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 常用
brew search 套件名
brew install 套件名
brew uninstall 套件名
brew list                    # 已裝
brew update                  # 更新 index
brew upgrade                 # 升級全部
brew cleanup -s              # 清理舊版本
brew autoremove              # 刪冇用依賴
brew doctor                  # 檢查有冇問題
brew services list           # 睇服務
```

> 💡 Homebrew 同時管理「formula」（命令）同「cask」（GUI app，如 Google Chrome）。

---

## Part 8：自動化

### 8.1 alias（快捷命令）

```bash
# ~/.zshrc 加
alias ll='ls -la'
alias cls='clear'
alias gs='git status'
alias ..='cd ..'
```

```bash
source ~/.zshrc   # 生效
```

### 8.2 寫 script

```bash
nano myscript.sh
```

```bash
#!/bin/bash
echo "開始..."
df -h
echo "完成"
```

```bash
chmod +x myscript.sh
./myscript.sh
```

### 8.3 排程（時間到自動做）

```bash
# cron 格式：分 時 日 月 星期
# 每朝 9 點
(crontab -l 2>/dev/null; echo "0 9 * * * /path/to/myscript.sh") | crontab -

crontab -l   # 睇
crontab -e   # 編輯
```

### 8.4 macOS 原生排程（launchd）

```bash
# 睇用戶層 daemon/agent
ls -la ~/Library/LaunchAgents/
launchctl list
```

---

## Part 9：救援技巧

### 9.1 冇空間（連開機都慢）

```bash
# 即時清到最多空間
rm -rf ~/Library/Caches/*
rm -rf ~/.Trash/*

# 睇最肥
du -sh ~/Library/* | sort -rh | head
```

### 9.2 DNS 出問題（上唔到網但 Wi-Fi 正常）

```bash
# 清 DNS cache
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder

# 測試
dig example.com +short
ping -c 3 example.com
```

### 9.3 App 開唔到（有權限問題）

```bash
# 清該 app cache（App 名改返）
rm -rf ~/Library/Caches/App名
# 或重設
defaults delete com.example.app 2>/dev/null
```

### 9.4 搵邊個 app 食緊某 port

```bash
lsof -i :8080
# 或者
sudo lsof -iTCP -sTCP:LISTEN -P | grep 8080
```

### 9.5 系統死機／狂轉沙灘波

```bash
# 強制退出 app
killall -9 "AppName"

# 如果 Finder 死
killall Finder

# 終極：如果成部機冇反應
# 長按電源鍵 10 秒強制關機（macOS 開返冇損壞）
```

---

## 附錄：最常用 Top 30 命令速查

| 命令 | 作用 |
|------|------|
| `ls -la` | 詳細列檔案 |
| `cd ~/path` | 轉資料夾 |
| `pwd` | 喺邊 |
| `cat` / `less` | 睇檔案 |
| `open .` | 開 Finder |
| `mkdir` / `touch` | 建資料夾/檔案 |
| `cp` / `mv` / `rm` | 複製/移動/刪除 |
| `find` / `mdfind` | 搵檔案 |
| `grep` | 搜尋內容 |
| `curl` | 下載/API |
| `ping` / `dig` | 網絡測試 |
| `ps` / `top` / `htop` | 睇進程 |
| `kill` | 殺進程 |
| `df -h` / `du -sh` | 睇磁碟/資料夾 |
| `chmod` / `chown` | 權限 |
| `brew install` | 裝套件 |
| `sudo` | 管理員執行 |
| `history` | 歷史命令 |

---

## 常見問題 FAQ

**Q：點樣知道一個命令係咩意思？**
A：`man 命令名`（說明書），或者 `命令名 --help`。例如 `man ls`。

**Q：打錯命令整壞咗嘢點算？**
A：視乎係咩：
- 刪錯檔案 → 睇 Trash（`open ~/.Trash`）
- 改錯 config → 通常有備份或者還原檔
- 權限搞錯 → `sudo chown -R $(whoami) ~/path`

**Q：點解要 sudo？**
A：系統級操作（改 /etc、裝系統套件）要管理員權限。`sudo` 會問你密碼。

**Q：Terminal 同 iTerm2 有咩分別？**
A：Terminal 係 macOS 內置。iTerm2（免費）功能多啲：分割視窗、更靚配色、profile。新手用內置 Terminal 已經夠。

**Q：zsh 同 bash 有咩分別？**
A：macOS 預設 zsh。大部分命令兩個都通用。寫 script 如果指明 `#!/bin/bash` 就係 bash。

**Q：點睇命令歷史／重跑？**
A：`history` 睇，`!!` 重跑上一個，`!123` 重跑第 123 個，`Ctrl+R` 反嚮搜尋。

---

## 總結

終端學嘅唔係「背命令」，係**「識用工具解決問題」**：

1. **初階**：`ls` / `cd` / `open` / `cat` — 日常生活
2. **中階**：`curl` / `grep` / `ps` / `brew` — 開發工作
3. **進階**：`alias` / script / cron — 自動化

**練習建議**：每星期學 5 條新命令，搵嘢做嗰時就用終端做一次（開檔、睇空間、測網絡）。一個月後你就會發現，好多嘢用終端其實快過用滑鼠。

**最緊要記住**：`rm -rf` 同 `sudo` 都要三思而後行，其他命令任你玩，玩壞咗最多重裝。