# FreshRSS AI 邮件简报

读取 FreshRSS 最近的未读文章，通过 OpenAI 兼容接口生成中文精选摘要，将 Markdown 在内存中转换为 HTML 邮件正文，再用 SMTP 发送。邮件包含原文链接，并附带纯文本备用正文。支持本地运行和 GitHub Actions 每日定时运行。

## 使用前准备

- Node.js 24 或更高版本。
- FreshRSS 已开启「允许 API 访问」，并在「个人资料」设置 API 密码。`FRESHRSS_API_PASS` 填这个密码，不是网页登录密码。详见 [FreshRSS API 配置](https://freshrss.github.io/FreshRSS/en/users/06_Mobile_access.html)。
- 可用的 OpenAI 兼容模型接口，以及支持 SMTP 的邮箱账号。SMTP 密码可能需要使用邮箱服务商的授权码或应用专用密码。
- GitHub 托管 runner 必须能连接 FreshRSS、模型接口和 SMTP 服务；仅本机或内网可访问的地址不能直接用于该 workflow。

## 在 GitHub Actions 运行

1. 将项目推送到 GitHub 仓库的默认分支，包含 `package.json`、`package-lock.json` 和 `.github/workflows/`。不要上传 `.env` 或 `node_modules`，这些已加入 `.gitignore`。
2. 打开仓库 **Settings → Secrets and variables → Actions → Secrets**，添加以下 Repository secrets：

| Secret | 内容 |
| --- | --- |
| `FRESHRSS_FEVER_URL` | Fever API 完整地址，例如 `https://rss.example.com/api/fever.php` |
| `FRESHRSS_USER` | FreshRSS 用户名 |
| `FRESHRSS_API_PASS` | FreshRSS API 密码 |
| `LLM_API_KEY` | 模型接口密钥 |
| `SMTP_HOST` | SMTP 服务器域名，例如 `smtp.example.com` |
| `SMTP_USER` | SMTP 登录账号，通常为发件邮箱 |
| `SMTP_PASS` | SMTP 密码或授权码 |
| `EMAIL_TO` | 收件邮箱；多个地址用英文逗号分隔，不带显示名称 |
| `EMAIL_FROM` | 可选，发件邮箱；默认使用 `SMTP_USER` |

3. 如需调整参数，在同一页面的 **Variables** 中添加 Repository variables：

| Variable | 默认值 | 说明 |
| --- | --- | --- |
| `LLM_BASE_URL` | `https://api.openai.com/v1` | 接口基础地址，不包含 `/chat/completions` |
| `LLM_MODEL` | `gpt-4o-mini` | 服务商支持的模型名 |
| `SMTP_PORT` | `465` | SMTP 端口 |
| `SMTP_SECURE` | 根据端口选择 | `465` 默认 `true`；其他端口默认 `false`，并强制 STARTTLS |
| `TIMEZONE` | `Asia/Hong_Kong` | 邮件标题日期使用的时区 |
| `LOOKBACK_HOURS` | `24` | 仅保留最近多少小时内的文章，正整数 |
| `MAX_ARTICLES` | `300` | 最多拉取多少篇最新未读文章，正整数 |
| `SUMMARY_LENGTH` | `250` | 每篇文章提供给模型的正文摘要字符数，正整数 |
| `MARK_AS_READ` | `false` | `true` 时，在邮件成功提交后标记本次扫描文章为已读 |

常用组合是端口 `465` + `SMTP_SECURE=true`，或端口 `587` + `SMTP_SECURE=false`。后者会通过 STARTTLS 升级为加密连接，服务端不支持时会失败。发件邮箱须符合 SMTP 服务商的要求。参见 [Nodemailer SMTP 配置](https://nodemailer.com/smtp)。

4. 打开仓库 **Actions → FreshRSS Email Digest → Run workflow**，手动运行一次，查看执行日志和邮箱收件情况。完整有效配置且存在符合条件的未读文章时才会发送邮件。

之后默认每天香港／北京时间 **08:15** 运行。调整 `.github/workflows/digest.yml` 的 `cron` 即可修改时间：当前 `15 0 * * *` 使用 UTC，相当于 UTC+8 的 08:15。`TIMEZONE` 只影响邮件标题日期，不改变执行时间。

定时 workflow 必须位于默认分支，可能因 GitHub 负载延迟；公开仓库连续 60 天无活动会自动停用定时 workflow，届时需在 Actions 中重新启用。参见 [GitHub 定时触发说明](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule)。

推送和 Pull Request 只触发独立的 `CI` 离线测试，不会发邮件或调用真实接口。定时与手动简报使用同一个并发组，避免两次任务同时执行。

## 本地运行

```bash
npm ci
cp -n .env.example .env
```

按 `.env.example` 填写 `.env`。若已有 `.env`，只补齐 SMTP 和邮件配置；原有 `WEBHOOK_*`、`TG_*` 不再使用。系统环境变量优先于 `.env`，适用于 GitHub Actions Secrets。

```bash
# 离线测试，不需要真实密钥，不发送邮件
npm test

# 实际获取文章、调用模型并发送邮件
npm start
```

## 发送与重试

- 没有符合时间范围的未读文章时，正常退出，不调用模型、不发邮件。
- 只处理时间窗口内、详情获取时仍未读的文章；未来日期文章会被排除。
- 缺少配置、接口响应字段异常、模型返回空摘要／明确拒绝／未正常完成，或 SMTP 发送失败时，任务以非零状态退出，GitHub Actions 显示失败。
- 只有 SMTP 接受全部收件人后，才会执行可选的已读标记。SMTP 接受代表邮件已提交给服务器，不保证最终进入收件箱，还需留意退信和垃圾邮件。
- `MARK_AS_READ=true` 标记的是本次送入模型的全部文章，不只是 AI 最终选入简报的文章。发送失败或部分收件人被拒绝时，不执行已读标记。
- 每次标记后核对 FreshRSS 返回的未读列表。若状态没有更新或无法确认，会报告“邮件已发送，但已读状态未确认”，避免将认证成功误当作标记成功。
- `MARK_AS_READ=false` 会保留未读状态，因此重复执行或时间窗口重叠可能重复收到相同文章；部分收件人收到邮件后重试，也可能重复收到邮件。标记已读中途失败时，已完成的标记无法回滚。

FreshRSS 本身仍需配置订阅源的定时刷新；这个脚本只读取 FreshRSS 已抓取的文章。
