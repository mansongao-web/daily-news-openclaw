# daily-news-openclaw（WSL 原生安装版）

基于你已经在 **WSL** 里全局安装好 OpenClaw（`openclaw -v` → `2026.4.8`）的前提。
本方案不用 Docker，让 OpenClaw daemon 常驻在 WSL 里运行。

每天 **08:00 / 19:00**（Asia/Shanghai）自动抓国内热点 / 财经 / AI 行业 RSS，
用 Claude（Sonnet 4.5）摘要后推送到飞书群。

> ⚠️ **关于飞书推送**：OpenClaw 的 cron `--channel` 原生只支持 Telegram/Discord/电话号码，**不支持飞书**。
> 本方案里飞书推送是在 SKILL.md 里让 Agent 用 `curl` POST 到 Webhook 完成的——
> 等于 OpenClaw 负责「定时 + Agent 规划 + LLM 调用」，飞书推送自己包一层。

---

## 运行环境注意事项（WSL 专属）

⚠️ **WSL 不开启就不会跑 cron**。Windows 关机 / 重启 / 把 WSL 实例关掉后，
OpenClaw daemon 也会停。两个处理办法：

- **临时**：日常让 WSL 常开，配合 Windows「电源 → 永不睡眠」
- **推荐**：把整套迁到 VPS（Linux 下命令完全一致），或者配置 `wsl --shutdown` 触发器
  + Windows 启动任务（开机自动 `wsl -d Ubuntu -u <user> -- openclaw daemon start`）

如果只是想「有时候在用」，跑 GitHub Actions 版更省心；坚持本地跑就接着看下面。

---

## 0. 先确认几个命令

在 WSL bash 里：

```bash
openclaw -v               # 应输出 OpenClaw 2026.4.8
openclaw cron --help      # 后面要用的子命令
openclaw skills --help
openclaw daemon --help
```

把这三条 `--help` 的输出先过一眼，对照下文命令里的 flag。如果你的版本里 flag 名略有差异
（比如 `--name` 变 `--id`），以 `--help` 为准。

---

## 1. 飞书机器人 + OpenRouter Key

### 飞书
1. 目标飞书群 → 群设置 → 群机器人 → **添加机器人** → **自定义机器人**
2. 复制 **Webhook URL**（形如 `https://open.feishu.cn/open-apis/bot/v2/hook/xxxx`）
3. 建议勾选 **签名校验**，复制 secret

### OpenRouter
1. <https://openrouter.ai> 注册 → 充值 5-10 美元
2. Keys → Create Key → 复制 `sk-or-xxxx`

---

## 2. 首次 onboard（如果你还没做过）

```bash
openclaw onboard --install-daemon
```

这一步会：
- 在 `~/.openclaw/` 生成默认 `openclaw.json` 和目录结构
- 注册 systemd（或 WSL init）用户服务，让 daemon 可以常驻
- 交互式填 provider：选 **openrouter**，填 API Key，模型选 `anthropic/claude-sonnet-4.5`

验证一下：
```bash
openclaw gateway --port 18789 --verbose
```
看到 `gateway listening on :18789` 就通了，`Ctrl+C` 退出。

---

## 3. 把仓库里的配置部署到 `~/.openclaw/`

本仓库在 Windows 下的路径是 `D:\Work\daily-news-openclaw\`，在 WSL 里对应
`/mnt/d/Work/daily-news-openclaw/`。

```bash
cd /mnt/d/Work/daily-news-openclaw

# 备份默认配置
if [ -f ~/.openclaw/openclaw.json ]; then
  cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
fi

# 覆盖主配置
cp ./openclaw.json ~/.openclaw/openclaw.json

# 部署技能
mkdir -p ~/.openclaw/workspace/skills/daily-news
cp ./workspace/skills/daily-news/SKILL.md ~/.openclaw/workspace/skills/daily-news/SKILL.md

# 确认
ls -la ~/.openclaw/workspace/skills/daily-news/
```

---

## 4. 注入环境变量

`openclaw.json` 用了 `${OPENROUTER_API_KEY}` / `${FEISHU_WEBHOOK}` / `${FEISHU_SECRET}` 占位符，
OpenClaw 启动时从环境变量读取。

在 WSL 里写到 `~/.bashrc`（或 `~/.zshrc`）末尾：

```bash
cat >> ~/.bashrc <<'EOF'

# === OpenClaw daily-news-bot ===
export OPENROUTER_API_KEY="sk-or-xxxxxxxxxxxx"
export FEISHU_WEBHOOK="https://open.feishu.cn/open-apis/bot/v2/hook/xxxx"
export FEISHU_SECRET=""   # 没开签名校验就留空
export OPENCLAW_HOOK_TOKEN="change-me-to-any-string"
EOF

source ~/.bashrc
```

> 如果你用 systemd user service 管理 daemon，shell 环境变量传不进去。
> 这种情况改把变量写到 `~/.openclaw/env`（OpenClaw 一般会自动加载），
> 或者在 `openclaw.json` 里直接硬编码值（注意别 commit 到 git）。

---

## 5. 重载技能 & 注册 cron

```bash
# 确认 OpenClaw 扫到了新技能（skills 是自动扫描的，没有 reload 命令）
openclaw skills list | grep -i daily-news
# 或看详情：
openclaw skills info daily-news

# 早报
openclaw cron add \
  --name "daily-news-morning" \
  --cron "0 8 * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "执行 daily-news 技能，抓取 RSS 并推送早报到飞书"

# 晚报
openclaw cron add \
  --name "daily-news-evening" \
  --cron "0 19 * * *" \
  --tz "Asia/Shanghai" \
  --session isolated \
  --message "执行 daily-news 技能，抓取 RSS 并推送晚报到飞书"

openclaw cron list

# 调试：立即跑一次（不等 cron）
openclaw cron run --name daily-news-morning
```

**说明**：
- `--session isolated` = 每次独立会话，不污染你的常驻对话
- **不用** `--channel` / `--to` / `--announce`，因为这些只支持 Telegram/Discord/电话，飞书是在 SKILL.md 里由 agent 自己 `curl POST` 过去的
- 如果需要指定 agent，加 `--agent <id>`（多 agent 环境才需要）

---

## 6. 启动 daemon & 手动触发一次

```bash
openclaw daemon start
openclaw daemon status

# 立刻跑一次验证整条链路
openclaw agent --message "立刻执行 daily-news 技能，推送一次测试消息到飞书"
```

30-60 秒后飞书群应收到卡片消息。

---

## 7. 日常运维

### 看日志
```bash
tail -n 100 ~/.openclaw/logs/agent.log
tail -f  ~/.openclaw/logs/agent.log        # 实时跟随
```

### 改信息源 / 排版
直接改 `~/.openclaw/workspace/skills/daily-news/SKILL.md`。**技能是自动扫描的**，改完下次 agent 触发就生效；
想立刻验证：
```bash
openclaw cron run --name daily-news-morning
# 或直接发一条 agent 消息：
openclaw agent --message "执行 daily-news 技能，推送一次"
```

**改完记得同步回仓库**（`/mnt/d/Work/daily-news-openclaw/` 下那份），这样有 git 历史、换机器能还原：
```bash
cp ~/.openclaw/workspace/skills/daily-news/SKILL.md /mnt/d/Work/daily-news-openclaw/workspace/skills/daily-news/SKILL.md
cp ~/.openclaw/openclaw.json /mnt/d/Work/daily-news-openclaw/openclaw.json
cd /mnt/d/Work/daily-news-openclaw && git add -A && git commit -m "tweak news skill"
```

### 暂停 / 恢复
```bash
openclaw cron disable --name daily-news-morning
openclaw cron enable  --name daily-news-morning

# 其他可用命令
openclaw cron show  --name daily-news-morning     # 看配置
openclaw cron edit  --name daily-news-morning --cron "0 7 * * *"   # 改时间
openclaw cron rm    --name daily-news-morning     # 删除
openclaw cron runs  --name daily-news-morning     # 看运行历史
openclaw cron status                              # 调度器整体状态
```

### 升级 OpenClaw
```bash
pnpm add -g openclaw@latest    # 或 npm i -g openclaw@latest
openclaw daemon restart
```

---

## 8. 排查清单

**`openclaw skills list` 看不到 daily-news**
→ 确认文件路径 `~/.openclaw/workspace/skills/daily-news/SKILL.md`，文件名必须全大写 `SKILL.md`。
→ 看看是不是落到了别的 agent workspace 里（多 agent 环境）：`openclaw skills list --agent <id>`

**cron 不按点触发**
1. `openclaw daemon status` 确认 daemon 在线
2. `openclaw cron list` 输出每行应有 `Asia/Shanghai`
3. **最常见原因**：WSL 实例没开机。`wsl --list --running` 在 Windows 里查
4. Windows 睡眠也会让 WSL 挂起 → 电源 → 从不睡眠

**飞书群没收到消息，但 agent 日志显示执行了**
→ Webhook 返回非 0 `code`。最常见：
   - URL 写错
   - 开了签名校验但 `FEISHU_SECRET` 没设 / 设错
   - 卡片 markdown 太长被飞书拒收，改 SKILL.md 里"每栏最多 N 条"调小

**RSSHub 公共实例 429 / 超时**
→ 自部署：
```bash
docker run -d -p 1200:1200 --name rsshub diygod/rsshub
```
把 SKILL.md 里的 URL 换成 `http://localhost:1200/...`（WSL 里容器是可直接访问的）。

**daemon 重启后环境变量丢失**
→ systemd user service 不会读 `.bashrc`。把变量改写到 `~/.openclaw/env`（如果 OpenClaw 支持）
  或 systemd unit 文件的 `Environment=` 里，或者直接硬编码到 `openclaw.json`。

---

## 9. 换模型 / 换 provider

### 官方 Anthropic（需要境外网络）
编辑 `~/.openclaw/openclaw.json`：
```json
{
  "agent": { "model": "anthropic/claude-sonnet-4-5" },
  "providers": {
    "anthropic": {
      "baseUrl": "https://api.anthropic.com",
      "apiKey": "${ANTHROPIC_API_KEY}"
    }
  }
}
```
```bash
echo 'export ANTHROPIC_API_KEY="sk-ant-xxx"' >> ~/.bashrc && source ~/.bashrc
openclaw daemon restart
```

### 本地 Ollama（离线 / 省钱）
```json
{
  "agent": { "model": "ollama/llama3.1:70b" },
  "providers": {
    "ollama": { "baseUrl": "http://localhost:11434" }
  }
}
```

---

## 10. 目录结构

```
# 版本管理（进 git）——Windows 这边
D:\Work\daily-news-openclaw\              <=> /mnt/d/Work/daily-news-openclaw/
├── openclaw.json
├── workspace/skills/daily-news/SKILL.md
├── .env.example
└── README.md

# OpenClaw 实际运行时读取的目录——WSL 这边
~/.openclaw/                              (== /home/marvyngao/.openclaw/)
├── openclaw.json
├── cron/jobs.json
├── logs/agent.log
└── workspace/skills/daily-news/SKILL.md
```

工作流：**改本仓库 → `cp` 到 `~/.openclaw/` → `openclaw skills reload`**。
