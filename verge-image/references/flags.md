# 生图参数参考

权威永远是 `verge-cli help <命令>`。

## 全局 flag

在子命令前后都生效，`verge-cli --json balance` 与 `verge-cli balance --json` 等价。

| flag | 说明 |
| --- | --- |
| `--api-key KEY` | API Key；未给则依次回落 `$VERGE_API_KEY`、配置文件 |
| `--base-url URL` | 接口地址，缺 `/v1` 会自动补；回落 `$VERGE_API_BASE_URL`、配置文件，最后 `https://api.verge-ai.xyz/v1` |
| `--json` | 打印原始 API 响应体（pretty 过），而不是人类摘要 |
| `--timeout DURATION` | 单请求 HTTP 超时，默认 `10m`（长任务可能要几分钟） |
| `--retries N` | GET 与预签名 PUT 的重试次数，默认 2；create / prepare / submit / 上传失败上报等 POST **永不自动重试** |
| `--skip-validate` | 跳过客户端参数校验，交服务端判。一般不要用 |
| `-v, --verbose` | 把每个 HTTP 请求打到 stderr，排查自建部署路由用 |
| `-h, --help` | 打印帮助 |

## `verge-cli task create` —— 生图任务


```
usage: verge-cli task create <prompt> [flags]
```

| flag | 说明 |
| --- | --- |
| `-m, --model MODEL` | 模型 id，默认取配置文件，再回落 `gpt-image-2` |
| `-r, --resolution RES` | `1080p` \| `2k` \| `4k` |
| `-a, --aspect-ratio RATIO` | `1:1` \| `16:9` \| `9:16` \| `4:3` \| `3:4` |
| `-n, --n COUNT` | 张数，1–4，默认 1 |
| `--group GROUP` | 计费分组覆盖 |
| `-f, --file PATH` | 本地二进制参考图，可重复；`NAME=PATH` 命名后可在提示词里写 `[@NAME]` |
| `-u, --image-url URL` | 公网参考图 URL，可重复；`NAME=URL` 命名 |
| `--base64-file PATH` | base64 编码图片文件（data: URI 或裸 base64），可重复；`NAME=PATH` 命名。走 task create 三段式上传（解码后 PUT 二进制） |
| `--base64-data DATA` | 内联 base64 编码图片（data: URI 或裸 base64），可重复；`NAME=DATA` 命名。跟 `--base64-file` 等价，数据直接写在命令行 |
| `--prompt-file PATH` | 从文件读提示词，`-` 表示读 stdin；**位置参数已给时被静默忽略**。文件上限 1MiB 且必须是 UTF-8 文本 |
| `--wait` | 轮询到终态才返回 |
| `-o, --output DIR` | 把结果图下载到 DIR；**隐含 `--wait`** |
| `--prefix NAME` | 下载文件名前缀 |
| `--wait-timeout DUR` | 轮询等待上限，默认 `20m`；超时（出口码 9）不代表任务失败 |
| `--poll-interval DUR` | 初始轮询间隔，默认 `2s` |

提示词与 `--prompt-file` **同时给出时位置参数优先，文件被静默忽略**（命令行手打的词是最终意图，不是报错）；都不给才报 usage 错误。`--prompt-file` 读的文件上限 1MiB 且必须是 UTF-8 文本，超限或夹带二进制会直接报错。



只要存在 `-f`、`--base64-file` 或 `--base64-data`，CLI 就自动执行 `prepare → 预签名 PUT → submit`；base64 会先解码成图片二进制。没有本地/base64 引用时才直接 POST 创建任务，公网 URL 用 `-u` 随请求提交。agent 不直接调用三阶段内部接口。

prepare 与 submit 都携带同一组模型、分辨率、比例、张数和 group。prepare 成功后已经有 `task_id`；后续失败按 [errors.md](errors.md) 恢复。

## `verge-cli task get` —— 查询任务

```
usage: verge-cli task get <task_id> [flags]
```

| flag | 说明 |
| --- | --- |
| `--wait` | 轮询到终态 |
| `-o, --output DIR` | 下载结果图；还在跑就先等 |
| `--prefix NAME` | 下载文件名前缀，默认 task id |
| `--wait-timeout DUR` / `--poll-interval DUR` | 同上 |

HTTP 200 不代表成功，要看 `status`。`status=failed` 的任务出口码 8。

## `verge-cli download` —— 下载结果图

```
usage: verge-cli download <task_id> [flags]
```

| flag | 说明 |
| --- | --- |
| `-o, --output DIR` | 写入目录，默认 `.` |
| `--prefix NAME` | 文件名前缀，默认 task id |

会先重新查一次任务拿新链接，而不是用早先存下来的 URL。任务还没到终态时直接报错，不等待 —— 要等就用 `verge-cli task get <id> --wait -o DIR`。

## `verge-cli models` / `verge-cli quota`

```
verge-cli models [--all]                            # 这个 Key 实际可用的模型
verge-cli quota [-m MODEL] [-r RES] [-a RATIO] [-n N]   # 这么一次生成会预扣多少额度
```

`verge-cli models` 默认只列声明支持图像生成的模型，`--all` 列全部。`verge-cli quota` 不创建任务、不扣额度，走的是和真实提交同一套定价代码。

## 模型 × 分辨率

CLI 内置表（用于少跑一次往返就能发现的错），**权威是 `verge-cli models`**。维护注意：CLI 的 `KnownModels` 或服务端上新模型后，用 `verge-cli models` 的实际输出核对本表：

| 模型 id | 名称 | 支持分辨率 | 速度 | 说明 |
| --- | --- | --- | --- | --- |
| `gpt-image-2` | GPT Image 2 | 1080p, 2k, 4k | 中等 | 默认模型，质量最佳 |
| `gemini-3-pro-image-preview` | Nano Banana Pro | 1080p, 2k, 4k | 中等 | 质量最佳 |
| `gemini-3.1-flash-image-preview` | Nano Banana 2 | 1080p, 2k, 4k | 快 | 日常推荐，速度与质量均衡 |
| `gemini-3.1-flash-lite-image` | Nano Banana 2 Lite | **仅 1080p** | 极快 | 轻量预览，质量基础 |

- 模型 id **大小写敏感**，不做折叠。
- 未知模型放行（只在 stderr 警告一句），只有已知模型才校验分辨率 —— 服务端随时上新模型，硬拦会让 CLI 立刻过时。
- 宽高比是协议级封闭枚举：`1:1`、`16:9`、`9:16`、`4:3`、`3:4`。写别的本地就被拦（出口码 5）。
- **实际像素不一定精确等于这个比例**：`1080p` + `16:9` 实测出的是 **1920×1088**（1088 = 64×17，上游把高度对齐到 64 的倍数）。要校验结果尺寸就留容差，别写精确相等。
- 默认值：`gpt-image-2` / `1080p` / `1:1` / `n=1`。用 `verge-cli config set model|resolution|aspect-ratio` 可改默认。

## 参考图

上限 **7 张**，本地文件与公网 URL **合计**。

```bash
# 不命名：按传入顺序生效
verge-cli task create "融合这两张图的风格" -f ./a.png -f ./b.png --wait -o ./out

# 命名：在提示词里用 [@名称] 精确指代。本地图与公网 URL 可以在同一次任务里混用
verge-cli task create "把 [@角色] 放进 [@背景] 里，保持光照一致" \
  -f 角色=./char.png \
  -u 背景=https://example.com/bg.png \
  --wait -o ./out
```

- `-f` / `--file` 传本地文件，`-u` / `--image-url` 传公网 URL，`--base64-file` 读取 base64 文件，`--base64-data` 接受裸标准 base64 或 data URI；**都可重复、可以混用**，不能用逗号合并。
- 混合场景中，本地文件和 base64 解码结果走三段式直传，公网 URL 在 submit 中追加；CLI 自动完成全部步骤。
- 本地上传顺序固定为 `-f` → `--base64-file` → `--base64-data`，公网 URL 随后追加。跨类型不要靠「第几张」推断，最稳妥的做法是**全部命名**并用 `[@名称]` 指代。
- `名称=值` 是命名形式。路径/URL 只在第一个 `=` 左边不含 `/` 或 `\` 时才按命名解析。`--base64-data` 会先把完整参数当 base64/data URI 校验，所以数据末尾的 `=` / `==` 不会被误认成名称分隔符。
- 名称里不能有 `[` 或 `]`（提示词里已经由 `[@...]` 包住了）。
- 服务端按 `[@名称]` **精确匹配**。名字写错不报错，那张图静默不参与生成；CLI 会在发请求前比一遍并在 stderr 警告，但不拦。
- **同名是硬错误**（出口码 5），且跨 `-f`、`-u`、`--base64-file`、`--base64-data` 一起查重：两张同名图必然有一张永远选不中。
- 单张公网参考图上限 10MB。本地文件和 base64 解码结果受 **10 MiB 单文件硬上限**约束，超限时 CLI 自动重编码 JPEG 并在 stderr 警告；重编码丢弃透明通道和动图帧。参考图总数 7 张按四种输入类型**合计**计算（`too_many_uploads`）。
- 把 URL 传给 `-f` 会被明确拒绝，提示改用 `-u`。

## 立即等待还是稍后查询

| 情况 | 用法 | 为什么 |
| --- | --- | --- |
| 交互式生成并立即拿文件 | `task create --wait -o DIR` | CLI 自动轮询并下载；`-o` 本身也隐含 `--wait` |
| 要并发跑多个 / 批量脚本 | `task create` 不带 `--wait` | 立刻返回 task id，不挂住当前进程 |
| 稍后继续等待 | `task get <id> --wait -o DIR` | 继续查询同一个任务，不重新创建 |


## 结果与落盘

`-o DIR` 会把每张图写成 `PREFIX-1.ext`、`PREFIX-2.ext`……扩展名取自响应 `Content-Type`（png / jpg / webp / gif / avif / bmp，认不出则回落 `.png`）。

> 输出格式不固定：4k 分辨率通常出 `.jpg`（文件更大但细节更丰富），1080p 通常出 `.png`。CLI 以响应 `Content-Type` 为准决定扩展名，不要硬编码。

| 命令 | 默认 PREFIX |
| --- | --- |
| `task create` / `task get` / `download` | task id |

`--prefix NAME` 可覆盖。落盘路径打在 **stderr**（`saved ...`），`task create` / `task get` 的 stdout 是人类摘要；`verge-cli download` 的 stdout 是落盘路径列表。

下载不带 `Authorization`（那些是对象存储签名直链），并会检查 `Content-Type`：不是图片就报错，绝不把 XML 错误体当图片落盘。

## `--json` 里能拿到什么

原样透传服务端响应体，服务端新增字段立刻可见。常用字段：

```jsonc
// task create / task get / download
{ "task_id": "task_xxx", "status": "completed", "model": "gpt-image-2",
  "created_at": 1736899200, "completed_at": 1736899320, "quota": 4000,
  "data": [ { "url": "https://...", "revised_prompt": "..." } ],
  "error": { "code": "...", "message": "...", "param": "..." } }   // 仅 status=failed 时存在
```

- `quota` 在排队 / 处理中可能是 0，那不是免费，只是还没结算。
- `b64_json` 恒为空串，服务端不回图片 base64。
- 图片链接 7 天后失效，`cover_url` 1 天。

## 轮询

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| `--poll-interval` | `2s` | 初始间隔，之后按 1.5 倍增长 |
| （内置上限） | `15s` | 间隔增长封顶 |
| `--wait-timeout` | `20m` | 等待上限；带参考图的任务文档写明可能跑 3–5 分钟 |

轮询进度打在 stderr（形如 `  in_progress … 12s`），`--json` 下不打印。超时返回出口码 9，任务仍在跑。
