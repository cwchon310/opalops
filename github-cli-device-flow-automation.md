# 用 AppleScript 自動化 GitHub CLI 的 Device Flow 登入

> 公開技術文 · 2026-08-19 · 適用 macOS + Safari

## 問題

`gh auth login --web` 會印出一次性 code（格式 `XXXX-XXXX`），然後要人手喺瀏覽器輸入。喺無頭／自動化環境，或者想全程自動化時，呢一步好煩。

## 解法

用 `osascript` 控制 Safari 完成成個 device flow：

```bash
# 1. 啟動認證（背景執行）
nohup gh auth login -h github.com -p https --web --git-protocol https > /tmp/gh-auth.log 2>&1 &
CODE=$(grep -oE '[A-Z0-9]{4}-[A-Z0-9]{4}' /tmp/gh-auth.log)

# 2. Safari 開 device 頁面
osascript -e 'tell application "Safari" to activate'
osascript -e 'tell application "Safari" to open location "https://github.com/login/device"'
sleep 4

# 3. 揀已登入帳戶（跳過 account picker）
osascript -e 'tell application "Safari" to do JavaScript "document.querySelector(\"form\").submit(); true" in current tab of front window'
sleep 4

# 4. 填 code（GitHub 用 9 個 input，每個一格，code 含連字號）
osascript -e 'tell application "Safari" to do JavaScript "var c=\"'"$CODE"'\"; var i=0; for (var ch of c) { var el=document.getElementById(\"user-code-\"+i); if (el) { el.value=ch; el.dispatchEvent(new Event(\"input\",{bubbles:true})); } i++; } true" in current tab of front window'

# 5. 提交
osascript -e 'tell application "Safari" to do JavaScript "document.querySelector(\"input[name=commit]\").click(); true" in current tab of front window'
sleep 6

# 6. 撳 Authorize
osascript -e 'tell application "Safari" to do JavaScript "var bs=document.querySelectorAll(\"button\"); for(var i=0;i<bs.length;i++){ if(bs[i].innerText.trim().startsWith(\"Authorize\")){ bs[i].click(); break; } } true" in current tab of front window'
sleep 5

# 7. 確認
gh auth status
```

## 重點

1. **Code 格式**：gh 印 `XXXX-XXXX`（9 個字含連字號），GitHub 頁面有 9 個 input 框（`user-code-0`…`user-code-8`），逐字填入。
2. **Event 觸發**：直接 `el.value = ch` 唔夠，要 `dispatchEvent(new Event('input', {bubbles:true}))` 先會令頁面 JS 感知。
3. **Account picker**：若頁面出現「Continue as xxx」，直接 `form.submit()` 跳過。
4. **Authorize 按鈕**：確認頁個 button 文字係 `Authorize github`，用 `innerText.startsWith("Authorize")` 搵。
5. **Timeout**：`nohup` 起嘅 gh 等太耐會俾 shell session 收走，code 會失效。要喺 code 有效期（~15 分鐘）內完成。

## 權限 scopes

`gh auth login` 預設攞 `repo`（完全控制 repos）+ `read:org` + `gist`，足以做最高權限嘅自動化操作。

## 後記

- macOS 要開「輔助使用」權限俾 Terminal/opencode 先可以用 `System Events`。
- 若用 `keystroke` 模擬打字更穩陣（GitHub 輸入框會自動跳格），但 `do JavaScript` 方式唔使 focus 權限，適合 headless。