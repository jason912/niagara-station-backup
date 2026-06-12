# 🏗️ Niagara Station Auto Backup

> Generic Niagara Station backup tool — SCRAM-SHA-256 auth → download `.dist` → NAS archive → email delivery

Supports Niagara 4.10+ (including JACE-8000 and Niagarax).

---

## 工作原理

```
用户手工准备                   自动备份脚本执行
┌─────────────────────┐      ┌──────────────────────────┐
│                     │      │                          │
│  Station 的 Files    │      │  1. SCRAM 登录站         │
│  └── px/            │      │  2. 访问 backup.px       │
│      └── backup.px  │ ────→│  3. 提取 CSRF token      │
│          ↑ 手工创建  │      │  4. 点击 Backup Button    │
│                     │      │  5. 下载 .dist 备份文件   │
│ backup.px 内容:     │      │  6. 存本地 + NAS + GitHub │
│ 把 BackupService    │      │  7. 可选邮件发送          │
│ 组件拖到 PX 页面，   │      │                          │
│ 选择 backup view    │      └──────────────────────────┘
└─────────────────────┘
```

### 关键前提：用户需手工创建 backup.px

脚本的原理是**模拟用户在浏览器中的操作**——进入后台页面，找到备份按钮，点击下载。Niagarax 系统本身没有预置一个可直接 HTTP 访问的备份页面，所以必须在 Station 的 `Files/px/` 目录下手动创建一个 `backup.px` 文件，把 **BackupService** 组件拖进去，选择 **backup** view。

如果没有这个文件，脚本就无法触发备份。

---

## 预备工作（必做）

### 1. 登录 Niagara Workbench（或 Niagarax Web界面）

### 2. 找到 Files → px 目录

在 Workbench 的 Nav 树中：
```
MyStation (localhost)
 └── Files
      └── px/
```

或者在 Niagarax Web 后台：
```
导航 → Config → Files → px/
```

### 3. 新建 backup.px 文件

> ⚠️ **N4.14 及以下版本**：直接在 `px/` 目录下新建一个 PX 文件，命名为 `backup.px`
>
> ⚠️ **Niagarax (N4.10+)**：进入 `Config → Files → px/`，新建 PX 文件，命名为 `backup.px`

### 4. 将 BackupService 组件拖入页面

- 从 Palette（组件面板）找到 `BackupService`
- 拖到新建的 `backup.px` 编辑器窗口
- 在弹出对话框中选择 **backup** view（或 BackupManager view）
- 保存文件

> 最终效果：通过浏览器访问 `http://YOUR_STATION_IP/px/backup.px` 可以看到一个备份管理页面，上面有 **Start Backup** 按钮。

---

## Features

- ✅ SCRAM-SHA-256 3-step AJAX authentication (N4.10 / N4.14 / Niagarax)
- ✅ 访问 backup.px 页面 → 提取 CSRF token → 触发备份下载
- ✅ 下载 `.dist` 备份文件（单次获取最新备份）
- ✅ 本地 + NAS 双存储
- ✅ Git push 到 GitHub（三重保险）
- ✅ Email delivery with attachment (SMTP)
- ✅ Single-file script, zero external deps (Node.js built-ins + nodemailer)

---

## Quick Start

### 1. Clone / Download

```bash
git clone <your-repo-url>
cd niagara-station-backup
```

### 2. Install dependencies

```bash
npm install nodemailer
```

### 3. Configure environment

```bash
cp .env.example .env
```

Edit `.env` with your station and email credentials:

```ini
# Niagara Station
STATION_HOST=192.168.x.x
STATION_PORT=80
STATION_SSL=false
STATION_USER=admin
STATION_PASS=your_password
STATION_NAME=Your_Station_Name

# SMTP Email (for sending backups)
EMAIL_USER=your@email.com
EMAIL_PASS=your_smtp_password
EMAIL_HOST=smtp.exmail.qq.com
EMAIL_PORT=465
EMAIL_TO=your@email.com

# NAS backup path (optional)
NAS_PATH=\\\\nas-server\\share\\backup

# Local save directory (default ./backups)
SAVE_DIR=./backups
```

### 4. Run backup

```bash
# Full flow: backup + NAS + email
node niagara-backup.js

# Backup only, no email
node niagara-backup.js --no-email

# Custom save directory
node niagara-backup.js --dir /path/to/save
```

---

## Commands

| Command | Description |
|---------|-------------|
| `node niagara-backup.js` | Full backup with email |
| `--no-email` | Skip email sending |
| `--dir /path` | Custom save directory |
| `--dry-run` | Test authentication only |
| `--verbose` | Detailed logging |

---

## 完整自动化流程（本机执行）

```bash
cd C:\Users\Administrator\.openclaw\workspace\niagara-station-backup

# 备份 144 站
copy .env.144 .env
node niagara-backup.js --no-email

# 备份文件自动提交到 GitHub
git add -A
git commit -m "Backup IAQ_Cloud $(date +%%Y-%%m-%%d_%%H:%%M)"
git push
```

### 备份文件存储位置

| 级别 | 路径 |
|------|------|
| 💻 本地磁盘 | `backups/`（脚本同目录） |
| 🗄️ NAS | `\\192.168.2.155\temp\niagara-backups\` |
| ☁️ GitHub | https://github.com/jason912/niagara-station-backup |

---

## Scheduling

### Windows Task Scheduler

```
Trigger: Daily at 00:00, 12:00
Action:  node D:\scripts\niagara-backup.js
```

### Linux / macOS crontab

```cron
0 */12 * * * cd /home/user/niagara-station-backup && /usr/bin/node niagara-backup.js >> backup.log 2>&1
```

---

## Project Structure

```
niagara-station-backup/
├── niagara-backup.js     ← Main script (自动备份引擎)
├── .env.example          ← Config template (no secrets)
├── .env                  ← Your actual config (in .gitignore)
├── .env.112              ← 多站配置：112站
├── .env.144              ← 多站配置：144站 (IAQ_Cloud)
├── README.md             ← This file
├── backups/              ← 备份输出目录 (auto-created)
├── SKILL.md              ← OpenClaw AI 技能定义
└── .gitignore
```

---

## Tech Details

### Authentication Flow

```
POST /prelogin (username)
  → SCRAM step 1 AJAX (client-first-message)
  → SCRAM step 2 AJAX (client-final-message + proof)
  → Session cookie
  → GET /px/backup.px
  → Extract CSRF token
  → Trigger backup download (startBackup=true)
```

Niagara N4.10/Niagarax uses a **3-step XHR-based SCRAM**:
1. `action=sendClientFirstMessage` — send client nonce
2. Server responds with salt + iterations + server nonce
3. `action=sendClientFinalMessage` — send computed SCRAM proof
4. Server validates and redirects to set session cookie

### Backup File

`.dist` files are ZIP archives (PK header) containing:
- `niagara_home/` — platform files
- `niagara_user_home/stations/<Name>/shared/` — station shared data

> **Note:** `.dist` is a distribution backup. It does NOT include PX files created dynamically via StringToFile or Workbench.

---

## 常见问题

**Q: 访问 backup.px 出现 404？**
A: Check if the file was created in the correct location (`Files/px/backup.px`). The default Niagarax installation does not include this file — it must be created manually in Workbench.

**Q: 备份文件太大，邮件发不出？**
A: Use `--no-email` flag. 大文件备份自动存本地 + NAS + GitHub，不需要邮件。

**Q: NAS 连不上？**
A: 确保 NAS 已开机，检查网络连接。技能会自动尝试连接，失败会跳过 NAS 存储但不会中断备份。

**Q: 如何验证脚本能正常登录站？**
A: 先用 `--dry-run` 测试认证：
```bash
node niagara-backup.js --dry-run
```

---

## 联系作者

有问题或建议，欢迎邮件联系：
**jason.zhang@gline-net.com**

---

## License

MIT
