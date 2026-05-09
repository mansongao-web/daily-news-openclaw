---
name: daily-news
description: 抓取国内热点 / 财经 / AI 三类 RSS，汇总成飞书 Markdown 卡片并推送
---

# 每日新闻简报

## 触发方式
由 OpenClaw cron 调用，消息中包含"推送早报/晚报"关键字。
推荐 cron 表达式：`0 8 * * *` 和 `0 19 * * *`，时区 `Asia/Shanghai`。

## 目标
产出一张排版整洁的飞书卡片：
- 30 字以内「今日导读」
- 三栏分组：**国内热点 / 财经要闻 / AI 行业**
- 每栏最多 6 条，每条 ≤40 字摘要 + 原文链接

---

## 步骤

### 1. 抓取 RSS

使用 `exec` 工具调用 `curl -sS --max-time 15` 并发（或顺序）拉取下列 URL，
把返回的 XML（RSS 或 Atom）解析成条目。任一源失败都**继续执行**，不中断。

**国内热点（category=domestic）**
- https://rsshub.app/weibo/search/hot
- https://rsshub.app/zhihu/hotlist
- https://rsshub.app/baidu/realtime

**财经要闻（category=finance）**
- https://rsshub.app/cls/telegraph
- https://rsshub.app/wallstreetcn/news/global

**AI 行业（category=ai）**
- https://rsshub.app/jiqizhixin/latest
- https://rsshub.app/qbitai
- https://huggingface.co/papers/rss

每源取前 10 条，提取 `title` / `link` / `description`（或 `summary`）。

> 💡 如果 OpenClaw 的 `📰 blogwatcher` 技能已启用（`openclaw skills list | grep blogwatcher`
> 显示 `✓ ready`），优先用它——它自带 RSS/Atom 解析和去重逻辑，比 curl + 手工解析稳。

### 2. 去重

- 同 category 内：标题相似度 > 85% 视为同一条，保留信息量最大的
- 跨天去重（可选）：把推送过的链接追加到 `~/.openclaw/workspace/skills/daily-news/sent.jsonl`，
  下一轮抓到就过滤掉 24h 内重复项

### 3. 生成简报（内部 LLM 调用）

按下列规则组织 Markdown：

> 你是专业资讯编辑。规则：
> 1. 仅使用提供的条目，不得编造链接或事实
> 2. 按【国内热点】【财经要闻】【AI 行业】三栏分组，每栏最多 6 条
> 3. 每条格式：`• <≤40字摘要> [阅读原文](URL)`
> 4. 摘要要有信息密度，避免复述标题和空洞形容词
> 5. 开头输出一行「今日导读」（≤30 字）概括主线
> 6. 直接输出 Markdown，不要代码块包裹，不要额外说明

### 4. 推送飞书（`exec` + curl）

判断 slot：当前小时 < 12 取「早报」，否则「晚报」。

组装请求体：

```json
{
  "msg_type": "interactive",
  "card": {
    "config": { "wide_screen_mode": true },
    "header": {
      "title": { "tag": "plain_text", "content": "📰 每日资讯 · <slot>" },
      "template": "blue"
    },
    "elements": [
      { "tag": "markdown", "content": "<上一步 LLM 输出>" },
      { "tag": "hr" },
      { "tag": "note", "elements": [
        { "tag": "plain_text", "content": "🕐 更新于 YYYY-MM-DD HH:mm" }
      ]}
    ]
  }
}
```

若 `$FEISHU_SECRET` 非空，按飞书签名算法追加 `timestamp` / `sign`：
- `timestamp = 当前秒级 UNIX 时间戳`
- `string_to_sign = f"{timestamp}\n{secret}"`
- `sign = base64(HMAC-SHA256(key=string_to_sign.encode(), msg=b""))`

执行：
```bash
curl -sS -X POST \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD" \
  "$FEISHU_WEBHOOK"
```

检查返回体的 `code == 0` 代表推送成功。

### 5. 记录已推送链接

把这一轮条目的 URL + timestamp 追加到
`~/.openclaw/workspace/skills/daily-news/sent.jsonl`，每行一个 JSON 对象。

---

## 失败处理
- **任一 RSS 源**失败：跳过，继续处理其他源
- **LLM 调用**失败：重试 3 次，指数退避（2s / 8s / 30s）。最终失败发一条纯文本报警到飞书
- **飞书推送**失败：重试 2 次，仍失败则只写日志到 stderr（OpenClaw 会落盘到 agent.log）

## 所需环境变量

| 变量 | 说明 |
|------|------|
| `FEISHU_WEBHOOK` | 飞书机器人 Webhook URL（必填） |
| `FEISHU_SECRET` | 飞书签名 secret（可选，开了签名校验才需要） |
| `OPENROUTER_API_KEY` | 已经通过 `openclaw.json.providers` 注入，本技能无需直接读取 |
