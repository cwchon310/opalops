# Docker 家庭伺服器完全指南

從零開始在家用伺服器（舊筆電、Raspberry Pi、NUC）上部署 Docker 服務。

---

## 問題

家裡有閒置硬體，想架設自己的服務（NAS、媒體伺服器、反向代理、監控），但不知道從什麼開始。

---

## 診斷

確認硬體與作業系統：

```bash
# 作業系統
cat /etc/os-release

# CPU 與記憶體
lscpu
free -h

# 磁碟空間
df -h

# Docker 是否已安裝
docker --version
docker compose version
```

---

## 修復

### 1. 安裝 Docker

**Ubuntu / Debian：**

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 將當前用戶加入 docker 組
sudo usermod -aG docker $USER
newgrp docker
```

**macOS：**

```bash
brew install --cask docker
```

### 2. 建立 `docker-compose.yml` 模板

```yaml
version: "3.8"

services:
  # 反向代理
  traefik:
    image: traefik:v2.10
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro

  # 檔案儲存（NAS）
  nas:
    image: lscr.io/linuxserver/samba:latest
    container_name: samba
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Hong_Kong
    volumes:
      - /mnt/data:/data
    ports:
      - "139:139"
      - "445:445"
    restart: unless-stopped

  # 媒體伺服器（Jellyfin）
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Hong_Kong
    volumes:
      - /mnt/media/config:/config
      - /mnt/media/media:/media
    ports:
      - "8096:8096"
    restart: unless-stopped

  # 監控（Uptime Kuma）
  uptime-kuma:
    image: lscr.io/linuxserver/uptime-kuma:latest
    container_name: uptime-kuma
    environment:
      - TZ=Asia/Hong_Kong
    volumes:
      - /mnt/data/uptime-kuma:/app/data
    ports:
      - "3001:3001"
    restart: unless-stopped
```

### 3. 啟動所有服務

```bash
docker compose up -d

# 查看狀態
docker compose ps

# 查看日誌
docker compose logs -f
```

---

## 4. 反向代理與域名

若你有域名，可用 Traefik + Cloudflare 自動申請 Let's Encrypt SSL：

```yaml
# docker-compose.yml 裡加標籤
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.jellyfin.rule=Host(`jellyfin.yourdomain.com`)"
  - "traefik.http.routers.jellyfin.entrypoints=websecure"
  - "traefik.http.routers.jellyfin.tls=true"
  - "traefik.http.routers.jellyfin.tls.certresolver=le"
```

---

## 教訓

| 教訓 | 說明 |
|------|------|
| 永遠用 `PUID`/`PGID` 對應主機用戶 | 避免檔案權限問題 |
| 重要資料掛載到主機目錄 | 容器刪除不會遺失資料 |
| 設定 `restart: unless-stopped` | 重開機後自動恢復 |
| 定期備份 `/mnt/data` | 防止硬體故障資料遺失 |

---

## 參考資料

- [Docker 官方文件](https://docs.docker.com/)
- [LinuxServer.io Docker 映像](https://www.linuxserver.io/)
- [Traefik 反向代理](https://doc.traefik.io/traefik/)
