# DigitalPlat 自动续期 · 调度与保活仓库（公开）

本仓库只包含**定时工作流**和**运行时间戳**（防 GitHub 60 天无活动自动停用定时任务），不含任何业务代码与凭据。

- 业务代码位于**私有仓库**，工作流运行时通过 `PRIVATE_REPO_TOKEN` 拉取执行；
- 每次运行（无论成功失败）都会把时间戳写入 `digitalplat/last_run.txt` 并提交，保证本仓库持续有 commit（每月一次，间隔远小于 60 天阈值），定时任务永不休眠。

## 每月自动执行

每月 1 日北京时间 09:30（UTC 01:30）运行一次：检查所有账号域名的到期时间，到期前 120 天内自动免费续期 1 年，结果通过 Telegram 推送。

## 需要配置的 Secrets（全部在本仓库）

| Secret | 必填 | 说明 |
|--------|------|------|
| `PRIVATE_REPO` | ✅ | 私有仓库全名，如 `yourname/digitalplat-auto-private` |
| `PRIVATE_REPO_TOKEN` | ✅ | 细粒度 PAT：仅授权该私有仓库、权限 `Contents: Read-only` |
| `DIGITALPLAT_SESSIONS` | 推荐 | 预置会话（本地 `tools/login_get_cookie.py` 登录一次获取，每行一个按账号顺序），CI 免登录直接续期 |
| `DIGITALPLAT_ACCOUNTS` | 二选一 | 每行 `邮箱:密码`（多账号） |
| `DIGITALPLAT_EMAIL` + `DIGITALPLAT_PASSWORD` | 二选一 | 单账号 |
| `HYSTERIA2_URI` | 强烈建议 | hysteria2 节点链接，工作流自动起代理把出口换成该节点（数据中心 IP 会被 Cloudflare 403） |
| `DIGITALPLAT_TELEGRAM_BOT_TOKEN` / `DIGITALPLAT_TELEGRAM_CHAT_ID` | 可选 | 运行结果通知（兼容旧名 `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID`） |

可选 Variables：`DIGITALPLAT_RENEW_BEFORE_DAYS`（默认 120）、`DIGITALPLAT_HTTP_PROXY`（自备代理时使用）。

## 手动触发

Actions → DigitalPlat Auto Renew → Run workflow → 勾选 **dry_run** 可只检查不续期。
