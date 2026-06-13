# 本地运行指南

> 本指南说明如何在 **不修改仓库代码、不污染 git 历史、不泄露 secret** 的前提下，
> 在本地机器上运行 `daily_stock_analysis` 并把分析报告推送到 Telegram。

适合场景：开发调试、单点验证、本地 cron 定时任务。

---

## 1. 设计原则

| 关注点 | 处理方式 |
|---|---|
| **仓库零修改** | 全部依赖、配置、运行脚本都放在 gitignore 之外 |
| **Secret 不入文件** | key/token 只放在 `chmod 600` 文件里，由启动脚本运行时注入到 env |
| **配置可复现** | 任何机器只需 copy 两个目录即可恢复（不含 secret） |
| **不影响 CI/Actions** | 仓库内不新增 workflow、不改 `.env.example` |

---

## 2. 前置条件

| 依赖 | 版本 | 说明 |
|---|---|---|
| Python | 3.10+ | 系统 Python 即可，不需要 sudo |
| pip | 最新 | 通过 `python3 -m ensurepip` 或 `get-pip.py` 安装 |
| minimax 账号 | 任意 | 需要 Coding Plan 订阅 + 余额 |
| Telegram bot | 任意 | 已有 bot 即可，不必新建 |

**不需要** `apt install python3-venv` 等 root 权限操作。

---

## 3. 目录布局（关键）

```
~/
├── dsa-secrets/                                # Secret 文件（chmod 600 目录）
│   ├── minimax.key                             # minimax Coding Plan API key
│   ├── telegram-bot.token                      # Telegram bot token
│   └── telegram-chat.id                        # Telegram chat id（个人或群组）
│
├── .config/dsa/                                # 配置（chmod 700 目录）
│   └── .env                                    # 非敏感配置（chmod 600）
│
├── dsa/                                        # 项目仓库（git clone 出来）
│   ├── .venv/                                  # Python 虚拟环境（gitignored）
│   ├── run.sh                                  # 启动脚本（gitignored）
│   ├── reports/                                # 报告输出（gitignored）
│   ├── data/                                   # 数据库 / 缓存（gitignored）
│   └── logs/                                   # 运行日志（gitignored）
```

**Secret 目录完全在仓库外面**，即使 fork 仓库被推到公网也不会泄露。

---

## 4. 准备 Secret 文件

```bash
mkdir -p ~/dsa-secrets
chmod 700 ~/dsa-secrets

# minimax key（去 platform.minimaxi.com 拿 Coding Plan key）
echo "sk-cp-你的key" > ~/dsa-secrets/minimax.key

# Telegram bot token（@BotFather 给的）
echo "1234567890:AAH..." > ~/dsa-secrets/telegram-bot.token

# Telegram chat id（个人或群组，正数/负数皆可）
# 个人：@userinfobot 查；群组：负数开头如 -100xxxxxxxxx
echo "123456789" > ~/dsa-secrets/telegram-chat.id

chmod 600 ~/dsa-secrets/*
```

---

## 5. 写非敏感配置

`~/.config/dsa/.env`（**只放非敏感变量**）：

```ini
# 自选股（用你的实际持仓替换）
STOCK_LIST=TSLA,COPX,NLR,XLK,MU,ALLW,FIG

# 时区
TZ=Asia/Shanghai

# 调试（生产可注释掉）
LOG_LEVEL=INFO
```

```bash
mkdir -p ~/.config/dsa
chmod 700 ~/.config/dsa
# 用编辑器填好上面内容后
chmod 600 ~/.config/dsa/.env
```

---

## 6. 准备 run.sh

`run.sh` 是启动脚本，**运行时**从 `~/dsa-secrets/` 读 secret 注入到环境变量，
脚本文件本身不包含明文 key。

放置位置：项目根 `/home/admin/dsa/run.sh`

参考实现：见本文档同目录的 `run.sh.example`（如不存在请按以下要点自己写）：

要点：
- `set -euo pipefail` 严格模式
- 启动时 `tr -d '[:space:]' < $KEY_FILE` 读 key 到 shell 变量 `KEY`
- `export MINIMAX_API_KEYS=*** LLM_MINIMAX_API_KEY=*** TELEGRAM_BOT_TOKEN=*** TELEGRAM_CHAT_ID="..."`
- `export ENV_FILE=$HOME/.config/dsa/.env`
- `cd $PROJECT_DIR && exec venv/bin/python3 main.py "$@"`

把 `run.sh` 放在项目根目录后 `chmod 700`。

**注意**：`run.sh` 已在仓库 `.gitignore` 中（项目自带规则），不会被误提交。

---

## 7. 装 Python 依赖

```bash
cd ~/dsa
python3 -m venv .venv
.venv/bin/python3 -m ensurepip --default-pip 2>/dev/null || \
  curl -sS https://bootstrap.pypa.io/get-pip.py | .venv/bin/python3
.venv/bin/python3 -m pip install -q -r requirements.txt
```

**只用 venv 内部 pip**，**不要**用 `pip install`（Debian/Ubuntu 默认 PEP 668 锁）。

---

## 8. 第一次试跑

```bash
~/dsa/run.sh --dry-run --force-run --stocks TSLA
```

预期日志：
- `[run.sh] key 长度: 125`
- `[run.sh] Telegram token 长度: 46, chat id 长度: 9`
- `已配置 1 个通知渠道：Telegram`
- `已配置 MiniMax 搜索，共 1 个 API Key`
- `LLM channel 'minimax': 3 model(s), 1 key(s)`

如果看到 `未配置任何可用的 AI 模型`，说明环境变量没注入成功，检查 `run.sh` 的 export 行。

---

## 9. 真实跑 + 推 Telegram

```bash
# 单只股票（5-10 分钟）
~/dsa/run.sh --force-run --stocks TSLA

# 全部自选股
~/dsa/run.sh --force-run

# 只跑个股，不跑大盘复盘（更快）
~/dsa/run.sh --force-run --no-market-review --stocks TSLA
```

跑完 Telegram 会收到 Markdown 格式的"决策仪表盘"。

---

## 10. 定时任务

```bash
# 每天美股收盘后 30 分钟（北京时间 5:00 夏令时 / 4:00 冬令时）
crontab -e
```

```cron
# m  h  dom mon dow  command
30  4  *  *  1-5  /home/admin/dsa/run.sh --force-run --no-market-review >> /home/admin/dsa/logs/cron.log 2>&1
```

`--no-market-review` 跳过大盘复盘，**让单次任务 < 5 分钟**，避免 cron 超时。

---

## 11. 常见问题

### minimax key 提示 "invalid api key"
- 确认 key 是 **Coding Plan 订阅**生成的（`sk-cp-` 开头）
- 国内用户用 `https://api.minimaxi.com/v1`（`.com`），不是 `.io`

### minimax 搜索返回 0 条
- 周末/节假日正常，周五收盘后用 `--force-run` 跳过非交易日检查
- 搜索窗口 `NEWS_MAX_AGE_DAYS=3` 默认 3 天

### Telegram 推送失败
- 检查 chat id 是否带负号（群组 id 是负数）
- 检查 bot 是否被用户拉黑
- 启动时会显示 `Telegram 消息发送成功` 才算成功

### `--dry-run` 和真实跑的区别
- `--dry-run`：拉数据 + 跑搜索，**不**调 LLM，**不**推送
- 真实跑：完整链路，**会**调 M3 计费

### 为什么不用 `pip install` 直接装
- 系统 Python 受 PEP 668 保护
- 用 venv 隔离避免污染系统包

### 升级依赖
```bash
cd ~/dsa
.venv/bin/python3 -m pip install -U -r requirements.txt
```

### 查看历史报告
```bash
ls -lt ~/dsa/reports/
```

---

## 12. 故障排查

| 现象 | 排查 |
|---|---|
| `python: not found` | `run.sh` 里的 `exec` 行写 `python` 改成 `python3` 或绝对路径 `.venv/bin/python3` |
| `ModuleNotFoundError: dotenv` | 没装依赖，重新跑步骤 7 |
| `Key 长度: 0` | `~/dsa-secrets/minimax.key` 是空文件或权限错 |
| `未配置通知渠道` | 重新跑 `export TELEGRAM_BOT_TOKEN=*** 确认 |
| 推送延迟 > 5 分钟 | minimax M3 是 reasoning 模型，思考 + 生成耗时；换 M2.7-highspeed 提速 |
