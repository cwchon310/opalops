# GitHub 從零到實戰：Git + gh CLI 完全上手

> 公開技術文 · 2026-08-19 · 適用 Windows / macOS / Linux

## 前言

「有 code 就要上 GitHub」已經係常識，但好多新手唔知 Git 同 GitHub 係兩樣嘢：

- **Git**：本地版控工具（記錄你每次改動）
- **GitHub**：雲端存放 Git repo 嘅平台（同步、協作、分享）

呢篇由零開始，教你：

1. **裝 Git + 第一次設定**
2. **本機三個核心概念**（working tree / staging / commit）
3. **上傳 GitHub**（push / pull / clone）
4. **分支與合併**（多人協作基礎）
5. **gh CLI**（唔使出網頁，終端搞掂一切）
6. **實用技巧**（badges、README、releases、自動化）

照抄即用，唔使記死命令，理解原理自然記得。

---

## Part 1：安裝與第一次設定

### 1.1 安裝 Git

```bash
# macOS（Homebrew）
brew install git

# Ubuntu / Debian
sudo apt install git

# Windows
# winget install Git.Git  或去 git-scm.com 下載
```

### 1.2 第一次設定（一定要做）

```bash
git config --global user.name "你的名字"
git config --global user.email "you@example.com"

# 檢查
git config --list
```

> 💡 呢兩個設定會寫入每個 commit。GitHub 會用你個 email 將 commit 對應到你帳戶。

### 1.3 如果係 GitHub，建議埋裝 gh CLI

```bash
# macOS
brew install gh
# 其他平台去 github.com/cli/cli 睇安裝方法
```

登入（見 Part 5）：

```bash
gh auth login
```

---

## Part 2：三大核心概念（搞懂呢三個就唔使驚）

```
工作目錄（Working Tree）    暫存區（Staging Area）    提交（Commit）
你嘅檔案改動  ──git add──►  排緊隊嘅改動  ──git commit──►  永久記錄
```

**口訣：改 → add → commit → push**

- **Working Tree**：你硬碟上嘅檔案
- **Staging**：你話「呢啲改動準備好」嘅暫存區
- **Commit**：將暫存區「影相」，成為歷史記錄，有自己嘅 hash

### 第一次完整流程

```bash
# 1. 去你嘅專案資料夾
cd ~/projects/my-project

# 2. 初始化（第一次先需要）
git init

# 3. 睇下狀態（會話你知有咩未 add）
git status

# 4. 將所有檔案加入暫存
git add .
# 或者指定檔案
git add README.md

# 5. 提交（寫清改咗咩）
git commit -m "init: 加入 README"

# 6. 睇歷史
git log --oneline
```

---

## Part 3：上傳 GitHub（Push / Pull / Clone）

### 3.1 喺 GitHub 建一個空 repo

用網頁：New repository → 起名 → 唔好勾任何選項 → Create。

### 3.2 將本地 repo 推上去

```bash
# 加遠端（remote）
git remote add origin https://github.com/你帳戶/你repo.git

# 推上去（第一次要 -u 設定追蹤）
git push -u origin main
```

> 💡 而家 GitHub 預設分支叫 `main`（舊叫 `master`）。

### 3.3 以後日常流程

```bash
git add .
git commit -m "feat: 加咗新功能"
git push
```

### 3.4 攞人哋嘅 repo（Clone）

```bash
git clone https://github.com/某人/某repo.git
cd 某repo
```

### 3.5 更新本地（Pull）

```bash
git pull
```

### 3.6 取消／修正

```bash
git status                        # 睇狀態
git reset HEAD <file>             # 取消暫存
git checkout -- <file>            # 回復檔案到上次 commit
git commit --amend                # 修改上次 commit message
git reset --soft HEAD~1           # 取消上次 commit 但保留改動
```

---

## Part 4：分支與合併（多人協作核心）

### 4.1 點解要分支？

你唔會直接喺 `main` 上面亂改。正確流程：開個分支 → 改 → 測試 → 合併返 main。

```
main:  o---o---o---o
                 \
feature:          o---o---o   ← 你嘅新功能
```

### 4.2 基本操作

```bash
# 開新分支 + 跳過去
git checkout -b feature/new-button

# 改嘢 → add → commit（照常）

# 跳返 main
git checkout main

# 將 feature 合併入 main
git merge feature/new-button

# 刪分支
git branch -d feature/new-button

# 睇所有分支
git branch -a
```

### 4.3 合併衝突（撞咗點算）

兩邊改同一行就會衝突。Git 會將檔案標記：

```
<<<<<<< HEAD
你嘅版本
=======
人哋嘅版本
>>>>>>> feature/xxx
```

**做法**：手動揀定邊個 → 刪除 `<<<<<<<` 標記 → 儲存 → 再 commit。

```bash
git add .
git commit -m "merge: 解決衝突"
```

### 4.4 多人協作標準流程（重要！）

```bash
# 1. 開始前先 pull 最新
git pull --rebase

# 2. 開分支
git checkout -b feature/xyz

# 3. 做嘢 + commit

# 4. 推分支上去
git push origin feature/xyz

# 5. 喺 GitHub 開 Pull Request（PR）

# 6. 人哋 review + merge
```

---

## Part 5：gh CLI（終端搞掂 GitHub）

### 5.1 登入

```bash
gh auth login
# 揀 GitHub.com → HTTPS → 用瀏覽器授權
# 完成後：
gh auth status
```

### 5.2 建 repo + 上傳（一條龍）

```bash
# 喺本地 repo 內
gh repo create my-repo --public --source . --push
```

一次過：建 GitHub repo → 加 remote → push。

### 5.3 常用 gh 命令

```bash
gh repo view                # 睇 repo 資訊
gh repo view --web          # 開網頁
gh issue list               # 睇 issues
gh issue create -t "標題" -b "內容"
gh pr create --fill         # 開 PR
gh pr list                  # 睇 PR
gh release create v1.0.0 --generate-notes   # 發佈 release
gh api repos/你帳戶/你repo   # 直接叫 GitHub API
```

> 💡 任何 GitHub 網頁功能，搵唔到 gh 命令就 `gh api` 直接叫 API，萬能。

### 5.4 檢查 GitHub 上嘅資料

```bash
# 睇 repo 所有檔案
gh api repos/你帳戶/你repo/contents/ --jq '.[].name'
```

---

## Part 6：實用技巧（令你嘅 repo 加分）

### 6.1 README badges（醒目標籤）

喺 README 頂部加：

```markdown
![License](https://img.shields.io/badge/license-MIT-blue)
![Platform](https://img.shields.io/badge/platform-macOS-9cf)
```

`shields.io` 有大量現成 badge 樣式，唔使自己整圖。

### 6.2 寫好 README 嘅結構

```markdown
# 專案名

> 一句話講清你做咩

## 功能
- ✅ ...
- ✅ ...

## 安裝
\```bash
命令
\```

## 用法
\```bash
命令
\```

## License
MIT
```

### 6.3 .gitignore（唔好亂推）

```bash
# 例：node_modules 永遠唔應該上 GitHub
node_modules/
.env
*.log
.DS_Store
__pycache__/
```

### 6.4 Releases（發佈版本）

```bash
gh release create v0.1.0 --title "v0.1.0" --notes "第一個版本"
# 之後可附檔案
gh release upload v0.1.0 ./build/myapp.zip
```

### 6.5 GitHub Actions（自動化）

喺 repo 建 `.github/workflows/ci.yml`，例如自動跑測試：

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "自動跑緊測試"
```

推上去之後，每次 push 都自動執行。

---

## 附錄：Git 速查表

| 想做咩 | 命令 |
|--------|------|
| 睇狀態 | `git status` |
| 暫存所有 | `git add .` |
| 提交 | `git commit -m "msg"` |
| 上傳 | `git push` |
| 更新 | `git pull` |
| 睇歷史 | `git log --oneline` |
| 開分支 | `git checkout -b 名` |
| 切分支 | `git checkout 名` |
| 合併 | `git merge 名` |
| 刪分支 | `git branch -d 名` |
| 睇差異 | `git diff` |
| 還原檔案 | `git checkout -- 檔` |
| 取消暫存 | `git reset HEAD 檔` |
| 克隆 | `git clone URL` |
| 睇遠端 | `git remote -v` |

---

## 常見問題 FAQ

**Q：push 時要我輸入 username/password 點算？**
A：用 gh CLI 登入後，`gh auth setup-git` 會用 token 做認證，以後唔使入密碼。

**Q：commit message 點寫好？**
A：用 Conventional Commits：
- `feat:` 新功能
- `fix:` 修正
- `docs:` 文件
- `refactor:` 重構
- `chore:` 雜項
例如 `feat: 新增登入功能`、`fix: 修正登入失敗 bug`。

**Q：點解 push 唔到，話「non-fast-forward」？**
A：遠端有更新但你冇 pull。先 `git pull --rebase` 再 `git push`。

**Q：唔小心 commit 咗密碼點算？**
A：立即改密碼＋刪 commit 歷史（用 `git filter-repo` 或重設 repo）。GitHub 亦會自動偵測掃描已知密碼。**教訓：`.env` 一定入 .gitignore。**

**Q：分支同 main 差太遠，merge 好痛苦？**
A：保持分支細、commit 密、定期 `git pull --rebase main`。大改動拆細。

---

## 總結

| 概念 | 一句話 |
|------|--------|
| Git | 本地版控，記錄每次改動 |
| GitHub | 雲端 repo 平台，同步＋協作＋分享 |
| Staging | add 之後嘅「排隊區」 |
| Commit | 影相，成為歷史 |
| Branch | 平行世界，唔好直接改 main |
| PR | 攞你個分支去合併嘅請求 |
| gh | 終端版 GitHub，唔使出網頁 |

**練習路徑（建議照住做一次）：**
1. 起個 `my-project` 資料夾，寫個 README
2. `git init` → `add` → `commit`
3. `gh repo create --source . --push`
4. 開分支改嘢 → PR → merge

做完呢四步，你已經完成 90% 日常需要嘅 Git 操作。其餘遇到先查表，唔使死記。