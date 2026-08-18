# Raspberry Pi 家庭伺服器陣列：零基礎從零到一手管理多部 Pi

> 公開技術文 · 2026-08-19 · Raspberry Pi OS (Debian Bookworm) 適用

## 前言

一部 Raspberry Pi 係玩具，三、四部夾埋就係「家庭私有雲」。呢篇由零開始教你：

1. **刷 SD 卡**（第一次著機）
2. **Headless 設定**（無螢幕都得）
3. **SSH 安全強化**（唔好裸奔）
4. **基本健康監控**（CPU/記憶體/溫度）
5. **常用工具安裝**（Docker、Samba、監控）
6. **多部 Pi 管理術**（一次過睇全部）

全部用命令，照抄即用。適合完全新手。

---

## Part 1：第一次著機（刷 SD 卡）

### 1.1 你需要嘅嘢

- Raspberry Pi（任何型號）
- 一張 SD 卡（建議 16GB 以上，Class 10）
- 讀卡器
- 電源線（5V / 2.5A 或以上，唔好求其用電話充電器）

### 1.2 用 Raspberry Pi Imager（最簡單）

官方工具：[Raspberry Pi Imager](https://www.raspberrypi.com/software/)（支援 Windows / macOS / Linux）。

```bash
# macOS 用 Homebrew 裝
brew install --cask raspberry-pi-imager
```

開啟後：
1. 揀 **Raspberry Pi OS**（揀 `Raspberry Pi OS Lite`——無桌面版，伺服器夠用又慳資源）
2. 揀 SD 卡
3. **重要**：撳一下齒輪圖示（進階選項）：
   - 開 SSH
   - 設 hostname（例如 `pi-server-1`）
   - 設用戶名＋密碼（唔好用 pi/raspberry！）
   - 設 Wi-Fi（如果唔用網線）
4. 寫入 → 插入 Pi → 著機

> 💡 Lite 版唔帶桌面，之後全部用 SSH 操控，慳記憶體又穩定。

---

## Part 2：Headless 設定（無螢幕管理）

### 2.1 搵到 Pi 嘅 IP

```bash
# 方法 1：路由器管理頁睇 DHCP 清單（最穩陣）
# 方法 2：用掃描工具
sudo arp-scan --local 2>/dev/null || arp -a

# 方法 3：如果 hostname 有 set，通常可以直接
ssh pi-server-1.local
```

### 2.2 第一次 SSH 登入

```bash
ssh 用戶名@pi-server-1.local
# 或者
ssh 用戶名@192.168.x.x
```

### 2.3 更新系統（第一次必做）

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

> 💡 Raspberry Pi OS 冇自動安全更新，自己定期行上面兩條命令。

---

## Part 3：SSH 安全強化（唔好裸奔）

以下全部喺 Pi 上執行。**唔做呢步，等同將門鎖打開。**

### 3.1 停用 root 登入

```bash
sudo nano /etc/ssh/sshd_config
```

改/確認以下設定：

```
PermitRootLogin no
PasswordAuthentication no    # 見 3.2 有 SSH key 先好改 no！
PubkeyAuthentication yes
```

### 3.2 設定 SSH 金鑰（唔使密碼登入）

喺你嘅電腦（Mac / PC）生成金鑰：

```bash
# 喺電腦上做一次
ssh-keygen -t ed25519 -C "yourname@laptop"
# 一路 Enter 就得（或者設 passphrase）
```

然後將公鑰 copy 去 Pi：

```bash
# 喺電腦上
ssh-copy-id 用戶名@pi-server-1.local
# 或者手動：
cat ~/.ssh/id_ed25519.pub | ssh 用戶名@pi-server-1.local \
  "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys"
```

測試金鑰登入成功後，先好去將 `PasswordAuthentication` 改做 `no`。

### 3.3 改 SSH 預設連接埠（可選，但推薦）

```bash
sudo nano /etc/ssh/sshd_config
# 改 Port 由 22 做 2222（或者你鍾意嘅數字）
Port 2222

sudo systemctl restart ssh
# 以後連接要用 -p
ssh -p 2222 用戶名@pi-server-1.local
```

### 3.4 防火牆（ufw）

```bash
sudo apt install ufw
sudo ufw default deny incoming
sudo ufw allow 22/tcp        # 或者你改咗嘅 port
sudo ufw allow 5353/udp      # 如果你要用 .local 解析
sudo ufw enable
sudo ufw status
```

---

## Part 4：基本健康監控

### 4.1 一眼睇狀態

```bash
# CPU + 記憶體
htop    # 或 top

# 溫度（Pi 最怕過熱！）
vcgencmd measure_temp

# 電壓／降頻
vcgencmd get_throttled
# 0 = 正常；0x50000 以上 = 有過熱降頻

# 磁碟
df -h

# 網絡
ip a
```

### 4.2 長期監控：用現成工具

**方案 A：neofetch + 自訂命令**

```bash
sudo apt install neofetch
neofetch
```

**方案 B：netdata（網頁儀表板）**

```bash
curl -Ss https://get.netdata.cloud/kickstart.sh | bash
# 完成後開瀏覽器去 http://pi-server-1.local:19999
```

實時 CPU / 記憶體 / 磁碟 / 溫度 / 網絡流量，手機都睇到。

### 4.3 自動檢查 script（每小時行一次）

```bash
sudo nano /usr/local/bin/pi-health.sh
```

```bash
#!/bin/bash
TEMP=$(vcgencmd measure_temp | cut -d= -f2 | tr -d "'C")
THROTTLED=$(vcgencmd get_throttled)
LOAD=$(cat /proc/loadavg | awk '{print $1}')
DISK=$(df -h / | tail -1 | awk '{print $5}')
echo "$(date): temp=${TEMP} load=${LOAD} disk=${DISK} throttled=${THROTTLED}" \
  >> /var/log/pi-health.log
```

```bash
sudo chmod +x /usr/local/bin/pi-health.sh
# 每小時行一次
(crontab -l 2>/dev/null; echo "0 * * * * sudo /usr/local/bin/pi-health.sh") | crontab -
```

---

## Part 5：常用工具安裝

### 5.1 Docker（跑容器最方便）

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# 重新登入先生效
sudo systemctl enable docker
```

**測試**：

```bash
docker run hello-world
```

> 💡 Pi 係 ARM 架構，揀 image 時注意用 `arm64` / `armhf` 版本。大部分主流 image 都有。

### 5.2 Samba（共享檔案俾 Windows / Mac / 電視）

```bash
sudo apt install samba
sudo nano /etc/samba/smb.conf
```

底部加：

```ini
[share]
   path = /home/用戶名/share
   browseable = yes
   read only = no
   valid users = 用戶名
```

```bash
mkdir -p ~/share
sudo smbpasswd -a 用戶名
sudo systemctl restart smbd
```

Mac 連接：Finder → Cmd+K → `smb://pi-server-1.local/share`

### 5.3 監控工具包

```bash
sudo apt install htop iotop sysstat tmux
```

`tmux` 好重要：SSH 斷線都唔會殺死你嘅 session。

```bash
tmux new -s work      # 開新 session
# ... 跑長任務 ...
# Ctrl+b 然後 d = 脫離
tmux attach -t work   # 重新接返
```

---

## Part 6：多部 Pi 管理術

當你有一排 Pi（pi-server-1、pi-server-2、pi-server-4...），要一次過睇晒。

### 6.1 SSH config（唔使記 IP）

喺你部電腦 `~/.ssh/config` 加：

```
Host pi1
    HostName pi-server-1.local
    User 你的用戶名
    Port 22

Host pi2
    HostName 192.168.x.x
    User 你的用戶名
    Port 2222
```

之後直接用 `ssh pi1`、`ssh pi2`。

### 6.2 一次過跑命令（全部 Pi）

用 `tmux` 開多個窗格，或者簡單 for loop：

```bash
# 喺電腦上，對每個 host 都 ping 一下
for h in pi1 pi2 pi4; do
  echo "== $h =="
  ssh -o ConnectTimeout=5 $h "hostname; uptime; vcgencmd measure_temp"
done
```

### 6.3 統一健康報告 script

```bash
#!/bin/bash
# check-all-pis.sh
for h in pi1 pi2 pi4; do
  echo "========== $h =========="
  ssh -o ConnectTimeout=5 $h "uptime; free -h | grep Mem; vcgencmd measure_temp; df -h / | tail -1" \
    || echo "!! $h 連唔到"
done
```

跑一次就知邊部死咗、邊部過熱、邊部滿咗。

---

## Part 7：Pi 冇反應咗？救援流程

**連唔到 = 唔一定死機。** 按呢個次序排查：

```bash
# 1. ping 到唔到
ping pi1

# 2. ARP 睇有冇上線
arp -a | grep -i "b8:27\|dc:a6\|e4:5f"

# 3. 有冇可能係 power 唔夠（紅燈閃 = 電壓不足）
# 換返 5V/3A 正規電源試下

# 4. 溫度過熱關機
# 摸下機身，太熱就斷電 5 分鐘再開

# 5. 最後手段：拔 SD 卡，用讀卡器插入電腦
# 開到個 boot 分割區，喺 cmdline.txt 加：
#   quiet
# 再 boot，睇 HDMI 或者重新用 Serial console
```

---

## 總結對比

| 需求 | 工具 | 一句話 |
|------|------|--------|
| 首次裝機 | Raspberry Pi Imager | 勾「進階選項」開 SSH |
| 無螢幕管理 | SSH + .local | `ssh pi.local` |
| 安全 | SSH key + ufw | 唔好用密碼裸奔 |
| 監控 | netdata | 網頁實時儀表板 |
| 容器 | Docker | 跑服務超方便 |
| 檔案共享 | Samba | 全平台通 |
| 斷線保護 | tmux | 長任務唔怕斷 |
| 多機管理 | ~/.ssh/config | `ssh pi1` 即連 |

**核心心法：**
- 唔好用預設密碼
- 定期 `sudo apt full-upgrade`
- 過熱係 Pi 最大敵人（加散熱片）
- 多部 Pi 用 SSH config + 統一 script 管理，唔好一部部手動撳

呢套組合拳落場之後，三、四部 Pi 就變成你嘅私有雲基礎。想加服務（NAS、廣告封鎖、監控、VPN），都係同一套路：SSH 上去 → 裝 → 設 systemd → 完成。