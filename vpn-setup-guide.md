# WireGuard VPN 架設完全指南

在自己的伺服器上架設 WireGuard VPN，實現安全遠端連接家庭/公司網路。

---

## 問題

公共 WiFi 不安全，想從外部安全連接家裡的 NAS、伺服器或內部服務。

---

## 診斷

確認伺服器環境：

```bash
# 作業系統
cat /etc/os-release

# 公開 IP
curl ifconfig.me

# 防火牆狀態
sudo ufw status
# 或
sudo firewall-cmd --state
```

WireGuard 需要 UDP 端口開放（預設 51820）。

---

## 修復

### 1. 安裝 WireGuard

**Ubuntu / Debian：**

```bash
sudo apt-get update
sudo apt-get install -y wireguard
```

**CentOS / RHEL：**

```bash
sudo yum install -y epel-release
sudo yum install -y wireguard-tools
```

### 2. 生成密鑰對

```bash
cd /etc/wireguard
umask 077

# 伺服器密鑰
wg genkey | tee server_private.key | wg pubkey > server_public.key

# 客戶端密鑰
wg genkey | tee client_private.key | wg pubkey > client_public.key
```

### 3. 設定伺服器 (`wg0.conf`)

```ini
[Interface]
Address = 10.0.0.1/24
SaveConfig = true
PrivateKey = <server_private_key>
ListenPort = 51820
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# Client
PublicKey = <client_public_key>
AllowedIPs = 10.0.0.2/32
```

> 將 `eth0` 改為你伺服器實際的網路介面名稱（用 `ip addr` 查看）。

### 4. 啟動 WireGuard

```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

### 5. 開啟防火牆端口

```bash
# UFW
sudo ufw allow 51820/udp

# firewalld
sudo firewall-cmd --permanent --add-port=51820/udp
sudo firewall-cmd --reload
```

### 6. 客戶端設定

**macOS / iOS (WireGuard App)：**

```ini
[Interface]
PrivateKey = <client_private_key>
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <server_public_key>
Endpoint = <你的伺服器公開IP>:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

**Linux 客戶端：**

```bash
sudo wg-setconf wg0 <(cat client.conf)
sudo wg-quick up wg0
```

### 7. 驗證連線

```bash
# 伺服器查看對等體狀態
sudo wg

# 客戶端測試連線
curl -I https://your-home-nas.local
```

---

## 教訓

| 教訓 | 說明 |
|------|------|
| 密鑰權限必須 `600` | `umask 077` 防止密鑰洩漏 |
| `AllowedIPs = 0.0.0.0/0` 代表全流量經 VPN | 只想連內網可改為 `10.0.0.0/24, <家庭網段>` |
| `PersistentKeepalive = 25` | NAT 環境下保持連線暢通 |
| 伺服器需開啟 IP 轉發 | `sysctl -w net.ipv4.ip_forward=1` |

---

## 參考資料

- [WireGuard 官方文件](https://www.wireguard.com/)
- [WireGuard 安裝指南](https://www.wireguard.com/install/)
