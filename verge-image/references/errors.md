# 出错怎么办

## 出口码

CLI 的出口码刻意分得细，**按码分支，不要 grep 文案**（文案随部署语言和版本变）。

| 码 | 含义 | agent 该怎么做 |
| --- | --- | --- |
| 0 | 成功 | 继续 |
| 1 | 本地错误（没配 Key、读文件、写磁盘、连不上服务端） | 看 stderr 那一行。`no API key found` 见 [setup.md](setup.md)；其余多半是路径写错或网络不通 |
| 2 | 用法错误（flag 不存在、缺参数、子命令拼错） | **别猜参数**，跑 `verge-cli help <命令>` 看真实用法 |
| 3 | 认证失败（HTTP 401 / 403） | Key 错 / 过期 / 被禁用。停下来告诉用户去查 Key，不要重试 |
| 4 | 额度不足（HTTP 402） | 停下来告诉用户余额不够（可跑 `verge-cli balance` 给出数字）。**不要重试** |
| 5 | 参数不合法（HTTP 400 或本地校验拦下） | stderr 会说清哪个字段、合法值是什么。改参数再跑 |
| 6 | 被限流（HTTP 429） | 退避后重试。同一个 Key 的并发任务数有上限，先减并发 |
| 7 | 服务端错误（HTTP 5xx） | GET / 预签名 PUT 可按 `--retries` 重试；create / prepare / submit 等 POST 不自动重试。若 stderr 有 `task_id`，先查原任务 |
| 8 | 任务以 `status=failed` 结束 | **不是网络问题**。把 `error.code` 和 `message` 原样报给用户。是内容策略之类的就改提示词，**不要**原样重跑 |
| 9 | 等待超时 | **任务还在服务端跑，不是失败。** 用同一个 `task_id` 跑 `verge-cli task get <id> --wait -o DIR` 继续等。**绝不重新创建任务** |

如果 stderr 报 `unexpected response`，说明服务端返回了意外的响应结构。记录 `request_id` 和 `task_id`（如有）；有 `task_id` 就查询原任务。prepare 请求完全没有返回 task id 时 CLI 无法凭空恢复，且不要盲目重复可能产生副作用的 POST。

`8` 和 `9` 是最容易搞混的两个：HTTP 200 不代表成功（`status=failed` 才是任务失败），而超时只代表本地不等了。

### HTTP 状态码 → 出口码

agent 拿到 raw 错误体（没映射成 `error.code`）时，按 HTTP 状态分支：

| HTTP 状态 | 出口码 | 含义 |
| --- | --- | --- |
| 400 | 5 | 参数不合法（多半本地拦下了） |
| 401 / 403 | 3 | 认证失败（Key 错 / 过期 / 无权） |
| 402 | 4 | 额度不足 |
| 404 | 5 | 任务不存在（`task_not_found`） |
| 409 | 5 | 会话冲突（`upload_session_*` / `retry_upload_required` / `idempotency_conflict`） |
| 429 | 6 | 限流（含 `concurrency_limit_exceeded`） |
| 5xx | 7 | 服务端错误 |
| 200 但 `status=failed` | 8 | 任务失败（不是网络问题） |

## 任务状态机

```
uploading ─┐
queued ────┼──→ in_progress ──→ completed   ← 终态
submit_unknown ┘                └→ failed   ← 终态
```

- **只有 `completed` 和 `failed` 是终态。** 其他状态都要继续轮询。
- **未知状态一律当非终态**处理（服务端将来会加中间态），不要当成成功或失败。
- `submit_unknown` 表示提交结果短期未确认 —— 用原 `task_id` 继续查，**不要重新提交**。
- `uploading` 表示 prepare 已创建任务，但上传/submit 尚未完成；未提交任务由服务端超时清理。这不等于已经完成最终生图计费。

## 服务端稳定错误码

出现在 `--json` 的 `error.code`，或人类输出里 `failure:` 段的 `code` 行。**按 code 分支，不要按 message**。其中 `generation_*` 是服务端固定 wire code，只表示图片任务结果，不是 CLI 子命令。

| code | 含义 | 怎么处理 |
| --- | --- | --- |
| `content_policy_violation` | 提示词或参考图触发内容策略 | 改提示词。**原样重跑必然再失败** |
| `prompt_too_long` | 提示词超 3000 字 | 精简提示词 |
| `invalid_parameter` / `missing_parameter` / `invalid_request` | 参数非法或缺失 | 看 `param` 字段，改那一个参数 |
| `unsupported_model` | 该 Key 用不了这个模型 | 跑 `verge-cli models` 换一个 |
| `image_count_invalid` | 张数或 `imageCount` 不合法 | `-n` 取 1–4 |
| `too_many_uploads` | 参考图超过 7 张 | 减到 7 张以内 |
| `invalid_reference_url` / `reference_unavailable` | 公网参考图地址不可达 | 换可公开访问的直链（不是网页地址） |
| `reference_not_image` / `reference_invalid_image` | 参考图不是图片或已损坏 | 换文件 |
| `reference_too_large` / `reference_size_unknown` | 单张公网参考图超 10MB 或拿不到大小 | 压缩后再试，或改用 `-f` 本地上传 |
| `insufficient_quota` | 额度不足 | 停下告诉用户充值。**不要重试** |
| `invalid_api_key` / `token_expired` / `token_disabled` | Key 无效 / 过期 / 被禁用 | 停下告诉用户换 Key |
| `permission_denied` | 该 Key 无权访问 | 停下告诉用户 |
| `concurrency_limit_exceeded` | 并发任务数超限 | 退避重试，或减少并发 |
| `task_not_found` | task id 不存在或不属于这个 Key | 核对 id 与 Key，别重建任务 |
| `submit_confirmation_timeout` | 提交结果未确认 | 用 stderr 中已有的 task id 执行 `verge-cli task get <id>`，**不要重新提交** |
| `generation_timeout` / `generation_failed` / `generation_unavailable` | 上游生成超时 / 失败 / 暂不可用 | 可以重试一次；连续失败报给用户 |
| `upload_session_forbidden` / `upload_session_invalid` / `upload_context_mismatch` | 上传会话无效、过期或上下文不匹配 | 重新运行完整 `verge-cli task create` 获取新上传槽位；不要复用旧 PUT URL |
| `retry_upload_required` | 某个文件需要重传 | 重新运行完整 `verge-cli task create` 获取新上传槽位 |
| `internal_error` | 服务端内部异常（HTTP 500） | 重试一两次，仍失败就报给用户并附 `request_id` |
| `unknown_error` | 未分类生图失败（HTTP 502） | 联系 Verge 管理员；不要无限重试 |
| `query_data_error` | 账户/额度数据读取失败（HTTP 500） | 重试一两次；账务系统恢复后通常自愈 |

人类可读输出的末尾和错误信息里**总是**带 `request_id`（取自网关的 `X-Oneapi-Request-Id` 响应头），报给用户时一并带上 —— 排查线上问题只靠它。`--json` 是原样透传，不注入 request_id。

## stderr 快查表

| stderr 里出现 | 原因 | 怎么修 |
| --- | --- | --- |
| `no API key found` | 没配 Key | `export VERGE_API_KEY=sk-...`，详见 [setup.md](setup.md) |
| `command not found: verge-cli` | 二进制不在 PATH | 见 [setup.md](setup.md)，把 `verge-cli` 放进 PATH |
| `task task_xxx prepared (uploading N file(s))` | prepare 成功，task_id 已可用 | 正常进度信息；如果后续 submit 失败，用这个 task_id 查询任务 |
| `uploading ref-N (N/M)` | 正在上传第 N 张参考图到对象存储 | 正常进度信息，M 张共需传完才 submit |
| `flag provided but not defined: -prompt` | 用了不存在的 flag | 提示词是位置参数；跑 `verge-cli help task create` |
| `a prompt is required` | 没给提示词 | 提示词写成第一个位置参数 |
| 位置参数和 `--prompt-file` 都给了，但文件被**静默忽略**（stderr 无提示） | 位置参数优先是刻意设计 | 别两个都传：手打的位置参数才是最终意图，文件里写的不会生效 |
| `read prompt: ... over the 1MiB prompt-file limit` / `is not valid UTF-8 text` | 提示词文件超 1MiB 或不是 UTF-8 文本 | 精简文件、转成 UTF-8；不要把图片当提示词传 |
| `"https://..." is a URL, not a local path` | URL 传给了 `-f` | 公网 URL 用 `-u` |
| `value is neither a data: URI nor valid base64` / `decoded base64 data does not look like an image` | `--base64-data` 不是合法图片 base64 | 传裸标准 base64，或 `data:image/*;base64,...`；命名时写 `名称=数据` |
| `cannot be re-encoded` / `could not fit under 10 MiB` | base64 或本地图片超 10 MiB 且无法压缩 | 先在本地转换/压缩成可解码图片，再重新运行任务 |
| `duplicate reference name(s): X` | 两张参考图同名 | 每张给不同名字 |
| `reference "X" is never used in the prompt`（warning） | 提示词里没写 `[@X]` | 在提示词里加上 `[@X]`，否则那张图不参与生成 |
| `reference name "X" must not contain [ or ]` | 名字里写了方括号 | `-f X=...`，方括号只出现在提示词里 |
| `<模型> does not support "<分辨率>"; it supports ...` | 这个模型不支持该分辨率 | 换分辨率，或换支持它的模型，表在 [flags.md](flags.md) |
| `"<比例>" is not supported; use one of ...` | 宽高比不在封闭枚举里 | 只能是 `1:1` / `16:9` / `9:16` / `4:3` / `3:4` |
| `model "X" is not in this CLI's known model list`（warning） | 模型不在内置表里 | 不一定是错；跑 `verge-cli models` 确认这个 Key 能用 |
| `timed out after ... the task is still running` | 等待超时（出口码 9） | `verge-cli task get <id> --wait -o DIR` 继续等 |
| `task X is still queued; wait for it with ...` | 对未完成任务用了 `verge-cli download` | 改用 `verge-cli task get <id> --wait -o DIR` |
| `server returned application/xml instead of an image` | 图片签名链接已过期 | 用 `verge-cli download <task_id>` 重新取新链接 |
| `download ...: HTTP 403 (the signed URL may have expired)` | 同上 | 同上 |
| `prepare returned N upload slot(s) for M file(s)` | 服务端返回的上传槽位数与本地/base64 总数不匹配 | CLI 停止上传；保留错误中的 `task_id` 和 `--verbose` 输出并报障 |
| `upload: ...; task_id X already exists` | prepare 已成功，但预签名 PUT 失败 | submit 未执行；原任务最终超时。重新运行完整任务获取新槽位 |
| `submit: ...; task_id X already exists` | submit 失败或结果不明确 | **不要重复 submit**；执行 `verge-cli task get X` 查询原任务 |
| `could not report upload failure ...`（warning） | 上传失败后的诊断上报未成功 | 保留原始 PUT 错误；会话由服务端最终超时清理 |
| `aborted` | 收到 Ctrl-C | 无需处理 |
| `unexpected response: HTTP ...` | 2xx 响应不是预期 JSON/协议结构 | 记录 status、Content-Type、`request_id`；有 `task_id` 就查询该任务，没有则联系服务端/人工对账，不要猜 ID 或重建任务 |

## 给 agent 的总结

1. **只有 `completed` / `failed` 是终态**，其余继续轮询，未知状态当非终态。
2. **出口码 9 不是失败** —— 用同一个 `task_id` 继续查，绝不重新创建任务。
3. **`--retries` 只用于 GET 和预签名 PUT。** create / prepare / submit / 失败上报 POST 都不自动重试。prepare 后错误会带 `task_id`：PUT 失败重新运行完整任务，submit 不明确查询原任务，绝不重复 submit。
4. **出口码 3、4 直接停下告诉用户**，重试只会浪费时间和额度。
5. **出口码 8 报 `error.code`**，`content_policy_violation` 这类必须改提示词，原样重跑必然再失败。
6. **报错原样透传** `error.code` / `message` / `request_id`，不要自己编原因。
