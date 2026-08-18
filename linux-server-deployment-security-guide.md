# Linux 伺服器部署與安全完全指南：從裝機到防護一氣呵成

> 公開技術文 · 2026-08-19 · Ubuntu / Debian 系適用

## 前言

你有一部舊 PC 或者租咗一部 VPS，想將佢變成真正可用嘅伺服器。但 Linux 伺服器唔似 Windows 咁「裝完就用到」，要做得正確先唔會變成人哋嘅肉雞。

呢篇由零開始，分五大階段：

1. **裝機與基本設定**（Ubuntu Server 裝完第一件事）
2. **SSH 安全強化**（唔好裸奔上網）
3. **防火牆與網路防護**（ufw + fail2ban）
4. **服務管理**（systemd：唔好用 nohup 走天涯）
5. **監控與日常維護**（睇狀態、睇 log、自動更新）

所有命令以 Ubuntu 24.04 為準，Debian 基本通用。

---

## Part 1：裝機與基本設定

### 1.1 安裝 Ubuntu Server

- 去 [ubuntu.com/download/server](https://ubuntu.com/download/server) 下載 ISO
- 用 Rufus（Windows）/ balenaEtcher（macOS）寫入 USB
- 安裝過程會問你：
  - 用戶名（**唔好用 root**）
  - 主機名（例如 `server-01`）
  - **「OpenSSH server」一定要勾選！**
  - 唔使裝 Snap Store / 桌面版（Lite 精神）

### 1.2 裝完第一件事：更新

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y htop curl wget git unzip fail2ban ufw tmux
sudo reboot
```

### 1.3 設定時區

```bash
sudo timedatectl set-timezone Asia/Hong_Kong
timedatectl
```

### 1.4 主機名

```bash
sudo hostnamectl set-hostname server-01
# 或者編輯
sudo nano /etc/hostname
```

---

## Part 2：SSH 安全強化（最重要，冇得走位）

### 2.1 唔好用密碼登入，用 SSH 金鑰

喺你部電腦（Mac/Linux）生成金鑰：

```bash
ssh-keygen -t ed25519 -C "my-server-key"
# 一路 Enter 或者設 passphrase
```

傳上伺服器：

```bash
ssh-copy-id 用戶名@伺服器IP
# 或者手動
cat ~/.ssh/id_ed25519.pub | ssh 用戶名@伺服器IP \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

### 2.2 鎖死 sshd_config

```bash
sudo nano /etc/ssh/sshd_config
```

確認或修改：

```
# 唔准 root 登入
PermitRootLogin no

# 唔准密碼登入（確認金鑰 work 先改！）
PasswordAuthentication no

# 唔好接受空密碼
PermitEmptyPasswords no

# 唔好俾普通用戶變成 root（可選但推薦）
AllowUsers 用戶名
```

**重要次序**：先確認金鑰登入得先，先好改 `PasswordAuthentication no`。改錯鎖死自己，要入 recovery mode 先救得返。

```bash
sudo systemctl restart ssh
# 新開一個 terminal 測試登入成功先好關舊嗰個！
```

### 2.3 改 port（可選但推薦）

```bash
# sshd_config 入面
Port 2222
# 之後連接：
ssh -p 2222 用戶名@伺服器IP
```

### 2.4 二步驗證（TOTP，推薦進階）

```bash
sudo apt install libpam-google-authenticator
google-authenticator
# 跟住指示，用手機 Authenticator app 掃 QR
# 之後喺 sshd_config 加：
# ChallengeResponseAuthentication yes
```

---

## Part 3：防火牆與網路防護

### 3.1 ufw（Uncomplicated Firewall）

```bash
sudo apt install ufw
# 預設拒絕所有入，允許出
sudo ufw default deny incoming
sudo ufw default allow outgoing
# 放行 SSH（無論改冇改 port）
sudo ufw allow 22/tcp
sudo ufw allow 2222/tcp
# 如果要跑網頁
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
# 啟動
sudo ufw enable
sudo ufw status verbose
```

**記住：開機一定要 enable，唔好手動開住就算。**

### 3.2 fail2ban（自動封鎖暴力破解）

```bash
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

喺 `[sshd]` 區段確認／修改：

```ini
[sshd]
enabled = true
maxretry = 5
bantime = 3600
findtime = 600
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
# 會見到已封鎖嘅 IP 數
```

### 3.3 確認而家實際開咗咩 port

```bash
ss -tlnp
# 或者舊式
netstat -tulnp
```

**原則：淨係開你需要嘅。** 每開一個 port 都係多一個攻擊面。

---

## Part 4：服務管理（systemd）

### 4.1 點解唔好用 nohup？

```bash
nohup ./myapp &
# 缺點：
# 1. reboot 之後就冇咗
# 2. 唔會自動重啟
# 3. 冇 log 管理
```

要用 systemd。寫一個 service：

```bash
sudo nano /etc/systemd/system/myapp.service
```

```ini
[Unit]
Description=My Application
After=network.target

[Service]
User=用戶名
WorkingDirectory=/home/用戶名/myapp
ExecStart=/home/用戶名/myapp/start.sh
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp

# 常用命令
sudo systemctl status myapp        # 睇狀態
sudo systemctl restart myapp       # 重啟
sudo journalctl -u myapp -f        # 跟 log
```

**`Restart=always` 係重點**：crash 自動拉返起，伺服器先算「堅」。

### 4.2 睇資源（邊個食緊 CPU）

```bash
htop                      # 互動式，最推薦
top -bn1                  # 一次性輸出
ps aux --sort=-%cpu | head -10   # CPU Top 10
ps aux --sort=-%mem | head -10   # 記憶體 Top 10
```

---

## Part 5：監控與日常維護

### 5.1 自動安全更新（唔好手動追）

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
# 揀 Yes，之後系統會自動裝安全更新
```

### 5.2 睇 log

```bash
# 系統 log
journalctl -xe
# 開機 log
journalctl -b
# SSH 登入記錄
sudo journalctl -u ssh
# 或者
last
# 失敗登入
sudo grep "Failed password" /var/log/auth.log | tail -20
```

### 5.3 定期健康檢查 script

```bash
sudo nano /usr/local/bin/server-health.sh
```

```bash
#!/bin/bash
echo "=== $(date) ==="
echo "-- 負載 --"
uptime
echo "-- 記憶體 --"
free -h
echo "-- 磁碟 --"
df -h | grep -E "^/dev|Filesystem"
echo "-- 溫度(如有) --"
sensors 2>/dev/null | grep -i "core" || echo "no sensors"
echo "-- 最近登入 --"
last -5
```

```bash
sudo chmod +x /usr/local/bin/server-health.sh
# 每朝 8 點行一次
(crontab -l 2>/dev/null; echo "0 8 * * * /usr/local/bin/server-health.sh >> /var/log/server-health.log") | crontab -
```

### 5.4 磁碟空間警報

```bash
# 磁碟 > 80% 就發電郵（需要 mailutils）
df -h / | awk 'NR==2 {split($5,a,"%"); if(a[1]>80) print "磁碟滿咗: "$5 | "mail -s \"Disk Alert\" you@example.com"}'
```

---

## 附錄 A：Docker 部署（可選但推薦）

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

用 Docker 好處：環境隔離、升級簡單、同機跑多個服務互不干擾。

```bash
# 例子：直接跑一個 Nginx
docker run -d --name web -p 80:80 nginx
# 指定 restart 策略
docker run -d --restart unless-stopped -p 8080:80 nginx
```

## 附錄 B：VPS 常見檢查清單（新人買機後必做）

```bash
# 1. 改 root 密碼（如果俾你 root）
sudo passwd root

# 2. 新用戶（建議）
sudo adduser 用戶名
sudo usermod -aG sudo 用戶名

# 3. 傳金鑰（見 Part 2）

# 4. 鎖 SSH（見 Part 2）

# 5. 開 ufw + fail2ban（見 Part 3）

# 6. 開自動更新（見 Part 5）

# 7. 設時區 + hostname
```

---

## 常見問題 FAQ

**Q：改壞 sshd 鎖死自己點算？**
A：如果用 VPS，去供應商 web console（noVNC / Serial console）登入，改返 config 再 restart ssh。實體機就插返螢幕鍵盤入 recovery。

**Q：fail2ban 封咗自己 IP 點算？**
A：`sudo fail2ban-client unban <自己IP>` 或者等 bantime 過。如果係你部機個 IP 長期被 ban，檢查係咪暴力破解來源。

**Q：一定要用密碼認證做 SSH 咩？**
A：唔係，金鑰 + 唔好密碼係標準做法。密碼只係過渡用。

**Q：點知自己部機有冇被入侵？**
A：`last` 睇登入紀錄、`sudo grep "Failed password" /var/log/auth.log | tail -20` 睇暴力嘗試、`ss -tlnp` 睇有冇異常 port。定期做。

**Q：Windows 想 SSH 去 Linux？**
A：Windows 10+ 內置 OpenSSH client，直接喺 PowerShell 用 `ssh 用戶名@IP`。

---

## 總結對比

| 階段 | 工具/做法 | 一句話 |
|------|-----------|--------|
| 裝機 | Ubuntu Server + 勾 OpenSSH | 裝完先更新 |
| SSH | 金鑰 + 鎖 sshd_config | 唔好用密碼裸奔 |
| 網路 | ufw + fail2ban | 少開 port，自動封暴力 |
| 服務 | systemd + Restart=always | 唔好 nohup |
| 監控 | unattended-upgrades + health script | 自動更新、定期自查 |

**心法總結：**
1. **最小暴露**：淨係開你需要嘅 port
2. **唔用密碼登入**：金鑰先係正路
3. **自動化**：自動更新、自動重啟、自動封鎖
4. **留後路**：改 SSH config 前先確保有第二個方法入到機

做到呢四點，你部伺服器就算放喺公網，都唔會係砧板上嘅肉。