# DigitalPlat 免费域名自动续期（多账号模式）

DigitalPlat 免费域名（`.us.kg` / `.dpdns.org` / `.qzz.io` / `.xx.kg` ...）全自动续期：**只需控制台的账号密码，支持多账号**，每个账号自动登录 → 通过 API 检测域名是否可以续期 → 到期前 120 天窗口内自动免费续期 1 年 → Telegram 汇总报告。

对比参考项目 [nove886/digitalplat-renew-check](https://github.com/nove886/digitalplat-renew-check)（只做到期检查 + 提醒，续期要手动点按钮）：本项目把"续期"这一步也自动化了，且多账号一次跑完。

## 工作原理

```
对每个账号（互不影响，单个失败不阻塞其他账号）：
┌ 登录取得会话（三段式全自动）
│   a. 复用缓存会话 state/session_<账号哈希>.txt（仍有效直接用）
│   b. 使用预置会话 DIGITALPLAT_SESSIONS（CI 推荐）
│   c. 纯 HTTP 登录 POST /_panel_api/api/auth/login（空 turnstile_token，自动重试 3 次）
│   d. HTTP 被风控拒绝时 → 自动调起 tools/login_get_cookie.py
│      （真实 Chrome/Edge 打开登录页，无感 Turnstile 自动通过）
├ API 检测：GET /_panel_api/api/domains
│   到期时间 + 平台标记 can_renew；到期前 120 天内且可续期 → 续期队列
├ 续期：POST /_panel_api/api/domains/{域名}/renew
│   renewal_type=free, years=1（免费 +1 年），成功后回读核对新到期时间
└ 全部账号跑完 → Telegram 汇总通知（可选）
```

### 登录细节（为什么有三段式）

- 控制台登录接口接了 Cloudflare Turnstile。实测**浏览器里它是无感模式，会自动通过并填充 token**（家宽直连、数据中心代理都测通，无需任何人工点击）；
- 纯 HTTP 登录（token 留空）服务端会**按 IP/风险间歇性放行或强制**——被强制时返回 `Turnstile verification is required`，此时脚本自动切换到浏览器方式；
- 会话按账号独立缓存（文件名用账号 SHA-256 短哈希，不含明文邮箱），失效自动重新登录，全程无需人工抓 Cookie。

## 代理出站

设置 `DIGITALPLAT_HTTP_PROXY` 后，**所有出站流量**（API 请求、HTTP 登录、Telegram 通知、浏览器登录）统一走该代理：

```bash
DIGITALPLAT_HTTP_PROXY=http://127.0.0.1:8080    # hysteria2 客户端 http 入口
DIGITALPLAT_HTTP_PROXY=socks5://127.0.0.1:1080  # hysteria2 客户端 socks 入口
```

实测某数据中心 IP（hysteria2 乌克兰节点）出口下：登录页 200 无挑战、CSRF 端点 200、域名 API 与登录接口均正常到达（仅应用层 401/400），**Cloudflare 全程放行**。SOCKS 出站需要 `pip install pysocks`（requirements.txt 已含）。

## 快速开始

### 1. 本地运行（推荐先用 dry-run 验证）

```bash
pip install -r requirements.txt
pip install playwright && playwright install chromium   # 浏览器登录兜底用（一次性）

set DIGITALPLAT_ACCOUNTS=a@example.com:password1 b@example.com:password2
python digitalplat_auto_renew.py --dry-run   # 只检查不续期
python digitalplat_auto_renew.py             # 检查 + 续期
```

多账号也可以用账号文件（同格式，每行 `邮箱:密码`，`#` 注释）：`set DIGITALPLAT_ACCOUNTS_FILE=accounts.txt`。

### 2. 双仓库部署（私库放代码，公开库放工作流 + 时间戳）

**架构**：本仓库内容作为**私有仓库**（只放代码，不含工作流）；另建一个**公开仓库**放定时工作流 `auto-renew.yml` 和运行时间戳 `digitalplat/last_run.txt`。

- 定时任务跑在公开库 → 每次运行提交时间戳，公开库每月一次 commit（间隔约 31 天，远低于 60 天阈值），**永不触发 GitHub 的"60 天无活动停用"规则**；
- 公开库里没有任何业务代码与凭据；私库没有定时任务，也不需要保活提交。

**部署步骤**：

1. 新建**私有**仓库，上传本仓库全部文件（`digitalplat-auto-private.zip`，根层结构解压即得）；
2. 新建**公开**仓库，上传公开库内容（`digitalplat-auto-public.zip`，仅 `.github/workflows/auto-renew.yml`、`state/.gitkeep`、`README.md`）；
3. 在**公开库** Settings → Secrets and variables → Actions 添加：

   | Secret / Variable | 必填 | 说明 |
   |--------|------|------|
   | `PRIVATE_REPO` | ✅ | 私有仓库全名（`用户名/仓库名`） |
   | `PRIVATE_REPO_TOKEN` | ✅ | 细粒度 PAT：仅授权该私库、权限 `Contents: Read-only`（拉取代码用） |
   | `DIGITALPLAT_SESSIONS` | **CI 强烈建议** | 预置会话（见下），免登录直接续期 |
   | `DIGITALPLAT_ACCOUNTS` 或 `DIGITALPLAT_EMAIL`+`DIGITALPLAT_PASSWORD` | ✅ | 账号凭据 |
   | `HYSTERIA2_URI` | **强烈建议** | hysteria2 节点链接，自动起代理换出口 IP |
   | `DIGITALPLAT_BOT_TOKEN` / `DIGITALPLAT_CHAT_ID` | 可选 | 结果通知（兼容旧名 `DIGITALPLAT_TELEGRAM_BOT_TOKEN` / `DIGITALPLAT_TELEGRAM_CHAT_ID` / `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID`） |

4. 公开库 → Actions → DigitalPlat Auto Renew → Run workflow → 勾选 **dry_run** 首跑验证；
5. 之后每月 1 日北京时间 09:30 自动运行一次，进入 120 天窗口的域名自动续期。

> **PAT 创建**：GitHub → Settings → Developer settings → Fine-grained tokens → Generate new token → Repository access 选"Only select repositories"并只勾选私库 → Permissions 里 `Contents` 设为 `Read-only`（工作流只读代码，不写私库）。
>
> **CI 上最可靠的做法：预置会话**——平台风控只拦"登录"，不拦"已登录的会话"：
> 1. 本地运行一次 `python tools/login_get_cookie.py --email xxx --password xxx`（家宽 IP 下无感 Turnstile 自动通过）；
> 2. 把生成的 `cookies.txt` 内容填到公开库 Secret **`DIGITALPLAT_SESSIONS`**（多账号每行一个，按 `DIGITALPLAT_ACCOUNTS` 的账号顺序）；
> 3. Actions 每次运行先用会话直接续期（不再登录），会话失效时自动告警，本地重跑一次工具刷新 Secret 即可。
>
> **为什么还要配 `HYSTERIA2_URI`**：数据中心 IP 会被 Cloudflare 直接 403 挑战（实测确认），会话校验请求也需要能到达平台；配代理后出站走节点 IP，实测节点 IP 下 Cloudflare 全程放行。

### 3. 登录工具单独使用

```bash
python tools/login_get_cookie.py --email you@example.com --password xxx
# 可选：--proxy http://127.0.0.1:8080 或 socks5://127.0.0.1:1080
# 输出 cookies.txt（Cookie 头）+ storage_state.json
```

## 配置项

| 环境变量 | 默认 | 说明 |
|----------|------|------|
| `DIGITALPLAT_ACCOUNTS` | - | **多账号**，每行 `邮箱:密码`（`#` 注释，密码可含冒号） |
| `DIGITALPLAT_ACCOUNTS_FILE` | - | 账号文件（同格式，工作目录内） |
| `DIGITALPLAT_EMAIL` / `DIGITALPLAT_PASSWORD` | - | 单账号（无 ACCOUNTS 时使用） |
| `DIGITALPLAT_SESSIONS` / `DIGITALPLAT_SESSION` | - | **CI 推荐**：预置会话（按账号顺序 / 单账号） |
| `DIGITALPLAT_HTTP_PROXY` | - | **全局代理出站**（http/https/socks5） |
| `DIGITALPLAT_RENEW_BEFORE_DAYS` | `120` | 续期窗口（平台免费续期窗口即 120 天） |
| `DIGITALPLAT_RENEWAL_TYPE` / `DIGITALPLAT_RENEWAL_YEARS` | `free` / `1` | 免费续期固定值 |
| `RENEW_COOLDOWN_HOURS` | `24` | 同域名续期尝试冷却时间 |
| `SESSION_COOKIE_FILE` | 自动 | 会话缓存默认 `state/session_<账号哈希>.txt`（自动刷新，勿提交） |
| `DIGITALPLAT_BOT_TOKEN` / `DIGITALPLAT_CHAT_ID` | - | 可选通知（兼容旧名 `DIGITALPLAT_TELEGRAM_BOT_TOKEN` / `DIGITALPLAT_TELEGRAM_CHAT_ID` / `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID`） |
| `DRY_RUN` / `--dry-run` | - | 只检查不续期 |

## 安全设计

- 续期是变更操作：对超时/5xx 等**模糊结果不自动重试**，只告警人工复核；
- 平台答复"已续期/未到时间/不可续期"按**跳过**处理；`pendingdelete` 状态域名直接跳过；每域名 24h 冷却；
- 续期成功后**回读核对新到期时间**，未变化会在通知中标注"请人工确认"；
- 多账号互相隔离：单账号登录/续期失败不影响其他账号，结果汇总通知；
- 日志中邮箱脱敏（`ab***@domain`），状态文件以 SHA-256 短哈希为账号键，不落明文；
- 出站请求仅允许 http/https，校验目标主机拒绝本地/内网/保留地址；重定向逐跳校验；
- 账号密码只走环境变量/账号文件，不写日志；会话缓存与登录输出均已加入 `.gitignore`。

## 故障排查

| 现象 | 原因与处理 |
|------|-----------|
| `HTTP 登录不可用: Turnstile verification is required` | 平台按 IP/风险强制 Turnstile（间歇性）；脚本自动重试 3 次 + 浏览器兜底，CI 上通过率有限 → **本地跑一次登录工具、刷新 `DIGITALPLAT_SESSIONS`** |
| `浏览器登录失败: ...` | 无桌面环境（如 Actions）或 Chrome/Edge 未安装 → 本地运行，或确认代理/Chrome 安装正常 |
| `登录被拒: Incorrect E-mail or password` | 该账号密码错误；开邮箱 2FA 时请用有头模式在浏览器内完成验证 |
| `获取域名列表返回 HTTP 401/403` | 会话失效且自动重登失败 → 检查账号密码 / 刷新 `DIGITALPLAT_SESSIONS` / 删除 `state/session_*.txt` 重跑 |
| `复查到期时间未变化` | 平台异步处理 → 稍后到控制台确认，下次运行会自动再试 |
| Telegram 没收到消息 | 检查 `DIGITALPLAT_BOT_TOKEN` / `DIGITALPLAT_CHAT_ID`（兼容旧名）；未配置时只输出日志 |

## 免责说明

- 仅用于**自己账号**下域名的续期管理；请遵守 DigitalPlat 服务条款与接口频率限制（默认每月一次、每域名 24h 冷却），不要高频运行；多账号请确保均为本人资产。
- 接口为控制台同源私有 API（`/_panel_api/...`），平台可能调整——若字段/路径变化，跑一次 `--dry-run` 即可从日志定位问题。
- 感谢参考项目：[nove886/digitalplat-renew-check](https://github.com/nove886/digitalplat-renew-check)、[OUBIGFA/DigitalPlat-Domains-auto-renew](https://github.com/OUBIGFA/DigitalPlat-Domains-auto-renew)、[lbjxr/domains-renewal](https://github.com/lbjxr/domains-renewal)。

## 许可证

MIT
