# 排查 cc-connect 微信橋接「收唔到訊息」：一次 DNS 間歇性失敗的實戰

> 公開技術文 · 2026-08-19

## 背景

用 cc-connect 將 opencode 橋接到微信個人號，實行手機遙距控制本機 AI agent。某日開始，微信發訊息 bot 一直無反應，疑似「AI 發呆」。

## 診斷步驟

### 1. 確認進程存在

```bash
ps aux | grep -E "opencode|cc-connect" | grep -v grep
```

### 2. 睇 cc-connect log

```bash
tail -50 ~/.cc-connect/logs/cc-connect.log
```

關鍵線索——反覆出現：

```
level=WARN msg="weixin: getUpdates failed"
error="... dial tcp: lookup ilinkai.weixin.qq.com: no such host"
backoff=1s
```

即係 bot 喺「收訊息」呢一步就失敗咗，訊息根本入唔到 agent。**唔係 AI 發呆，係條橋斷咗。**

### 3. 現場測試 DNS

```bash
dig +short ilinkai.weixin.qq.com
```

十次測試十次都 resolve 到，但 log 就 98 次 `no such host` —— 典型**間歇性 DNS 失敗**。原因：系統用緊 Cloudflare (1.1.1.1) resolver，對騰訊呢啲域名時靈時唔靈（DNS 污染／劫持／resolver 不穩）。

### 4. 即場驗證系統 resolver

```bash
scutil --dns | grep nameserver
```

顯示 `1.1.1.1 / 1.0.0.1`。

## 修復

### 方案 A（推薦）：/etc/hosts 釘 IP

繞過 resolver，最直接：

```bash
# 先 dig 攞真 IP
dig +short ilinkai.weixin.qq.com
# 結果：43.163.165.187  43.163.179.90

sudo cp /etc/hosts /etc/hosts.bak
echo "43.163.165.187 ilinkai.weixin.qq.com
43.163.179.90 ilinkai.weixin.qq.com" | sudo tee -a /etc/hosts
```

確認：

```bash
dscacheutil -q host -a name ilinkai.weixin.qq.com
lsof -p <cc-connect-pid> | grep ESTABLISHED   # 應該見到 443 連線
```

### 方案 B：換穩陣 resolver

```bash
# 改用阿里 223.5.5.5 或騰訊 119.29.29.29
# macOS: 系統設定 → 網絡 → DNS
```

## 教訓

1. **「AI 唔答」唔一定係 AI 問題**——先查傳輸層：log 有冇 warning、訊息有冇入到。
2. **間歇性失敗最難捉**：log 累積 98 次先 reveal 到規律。一定要數頻率（`grep -c`）。
3. **DNS 污染對 bot 打擊特別大**：bot 靠長輪詢（getUpdates），每次 poll 都撞 DNS，一撞就斷一輪。
4. **釘 IP 要留意 IP 變更**：域名後端換 IP 要更新 /etc/hosts，最好定期 `dig` 對比。

## 類似場景

任何「bot／cron／長輪詢服務」對外網域名間歇性失敗，都可以用同一套排查法：log 先行 → 現場測試 → 釘 IP 或換 resolver。