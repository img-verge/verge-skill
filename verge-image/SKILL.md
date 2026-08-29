---
name: verge-image
description: 用 verge-cli 命令行工具调用 Verge 生成或编辑图片 (create or edit images with the Verge via verge-cli, text-to-image, image editing, reference images). 当用户要生成图片、文生图、出图、按参考图改图、合成、做海报、修图、风格迁移、二次编辑、批量出图、下载生成结果时使用。需要 verge-cli 可执行文件和 VERGE_API_KEY。
---

# Verge 图像生成

用 `verge-cli` 驱动 Verge 的图像接口出图。**适用 `verge-cli` ≥ v0.1.0**（`verge-cli version` 查看）。

## 预检

每次任务开始**先**做三件，再决定命令。**agent 不负责装 / 编译 `verge-cli`** —— 二进制是用户提前装好的。

1. **`verge-cli version`** —— 没装就停下，让用户去 [Releases](https://github.com/img-verge/verge-cli/releases) 下。完整流程见 [references/setup.md](references/setup.md)
2. **`verge-cli balance`** —— Key 没配 / 额度为 0 / 报 `no API key found` 都先停下。同样走 [references/setup.md](references/setup.md)
3. **批量任务**：再多一步 `verge-cli quota -m <模型> -r <分辨率> -n <总张数>` 估扣费，额度不够就告诉用户别继续

## 执行模型

- **只用 `verge-cli` 命令**，直接给 argv（别套 `sh -c`）。
- **默认用 `verge-cli task create --wait`**。自动处理本地文件、base64 和公网 URL 参考图；不要让 agent 自己调用 prepare、PUT 或 submit。
- **提示词超过 50 字或含中文/特殊字符时，优先用 `--prompt-file`**，避免 shell 编码问题。
- **默认参数**：模型 `gpt-image-2`、分辨率 `1080p`、宽高比 `1:1`、张数 1。
- **重试边界**：`--retries` 只用于 GET 和预签名 PUT；create / prepare / submit / 上传失败上报这些 POST 都不自动重试。

## 编辑 vs 生成

「编辑 / 改 / 修 / 转换 / 加元素 / 换背景」和「从零出图」走的是**同一条** `verge-cli task create --wait`，差别只在**提示词怎么写、参考图怎么用**：

| 意图 | 提示词长这样 | 参考图 |
| --- | --- | --- |
| 生成 | 描述新画面 | 可选，0–7 张 |
| 编辑 | 「把 [@x] 的颜色改成红色」 / 「把 [@x] 放进 [@y] 的背景里」 | 必传，1+ 张 |

没有专门「edit」命令。**靠参考图 + 提示词让模型知道要保留什么、改什么。**

## 决策表

| 用户意图 | 命令 |
| --- | --- |
| 文生图 | `verge-cli task create "提示词" --wait -o ./out` |
| 多张 / 指定参数 | `verge-cli task create "提示词" -n 2 -r 2k -a 16:9 --wait -o ./out` |
| 带 **本地文件**参考图 | `verge-cli task create "把 [@x] 做成海报" -f x=./a.png --wait -o ./out` |
| 带 **公网 URL** 参考图 | `verge-cli task create "背景换成 [@x]" -u x=https://... --wait -o ./out` |
| 带 **base64** 参考图 | `verge-cli task create "参考 [@x] 的风格" --base64-file x=./img.b64.txt --wait -o ./out` |
| 带 **base64 内联**参考图 | `verge-cli task create "参考 [@x]" --base64-data x='data:image/png;base64,...' --wait -o ./out` |
| 不想挂着等 | `verge-cli task create "提示词"` → 记 `task_id` → 之后 `verge-cli task get <id> --wait -o ./out` |
| 取已完成任务的图 | `verge-cli download <task_id> -o ./out` |
| 查模型 / 查余额 / 查额度 | `verge-cli models` / `verge-cli balance` / `verge-cli quota -r 2k -n 2` |

`-o DIR` 隐含 `--wait`：指定了下载目录就会自动等到出图。

多张参考图每张写一个 flag；`-f`、`-u`、`--base64-file`、`--base64-data` 合计最多 7 张。跨类型时全部命名并用 `[@名称]`，不要依赖位置。

**base64 选型**：内容在文件里（几百字符以上）→ `--base64-file 名称=./img.b64.txt`；短串可用 `--base64-data 名称=裸标准base64` 或 `名称=data:image/*;base64,...`。两者都会先由 CLI 解码、校验图片，再走三段式直传；末尾 `=` / `==` 不会被当成名称分隔符。

**批量前先估扣费**：`verge-cli quota -n <总张数> -m <模型>`，不够就停下来告诉用户，别做完才发现额度不够。

## 常见场景示例

完整的可复制命令块。目录要先 `mkdir -p` 建好。

**1. 文生图，单张 1080p**
```bash
verge-cli task create "雨夜霓虹街头，赛博朋克，电影感" --wait -o ./out
```

**2. 按参考图改图（本地文件）**
```bash
verge-cli task create "把 [@角色] 放进 [@背景] 里，保持光照一致" \
  -f 角色=./char.png -u 背景=https://example.com/bg.png \
  --wait -o ./海报
```

**3. 批量 4 张同主题，4k**
```bash
# 先估扣费
verge-cli quota -m gpt-image-2 -r 4k -n 4
# 再跑（-n 控制张数）
verge-cli task create "极简风格的山脉图标，纯色背景" -n 4 -r 4k --wait -o ./icons
```

**4. 长提示词写到文件**
```bash
cat > /tmp/prompt.txt <<'EOF'
A cinematic wide shot of a rain-soaked Tokyo backstreet at 3am,
neon signs reflecting on wet asphalt, lone figure under a red umbrella,
anamorphic lens flare, 35mm film grain, Blade Runner color palette.
EOF
verge-cli task create --prompt-file /tmp/prompt.txt -r 2k -a 16:9 --wait -o ./out
```

**5. 异步提交 + 稍后取**
```bash
# 提交，不挂等
TID=$(verge-cli task create "赛博朋克猫" --json | jq -r '.task_id')
echo "提交了 $TID，去忙别的"
# 之后
verge-cli task get "$TID" --wait -o ./out
```

## 硬规则

- **提示词是位置参数**，没有 `--prompt`。含换行时用 `--prompt-file`；位置参数和 `--prompt-file` 同时给时位置参数优先。
- **`[@名称]` 必须与 `-f` / `-u` / `--base64-file` / `--base64-data` 的名字逐字一致**。名字写错不会报错，但那张参考图不参与生成。
- 本地文件、base64 和公网 URL 参考图合计 ≤ **7** 张；`-n` 取 **1–4**；提示词 ≤ **3000 字**。
- 本地文件和 base64 解码结果超过 10 MiB 时 CLI 自动重编码 JPEG；会丢透明通道和动图帧。
- 并发生图任务 ≤ **5** 个，超限返回 `concurrency_limit_exceeded`（HTTP 429）。
- 本地文件按内容识别类型，不信任扩展名。
- **要文件必须给 `-o DIR`**。不给只打印 7 天有效的链接。
- **按出口码分支**（见 [references/errors.md](references/errors.md)），不要 grep 文案。特别 `9` 是等待超时，任务仍在跑。
- 失败时把 `error.code` 和 `message` 原样报给用户。prepare 已成功时 stderr 会带 `task_id`：上传失败则重新运行完整任务获取新槽位；submit 结果不明确则查询原任务，绝不重复 submit。

## 不要发明

### 子命令幻觉

agent 会从「`task create`」自然想到这些，但都不存在：

```
verge-cli image / verge-cli img              # 命令就叫 task create
verge-cli task submit ...                    # 内部步骤，CLI 不暴露
verge-cli upload / verge-cli task download   # 不存在；下载走 download <id>
```

### flag 错值

```
verge-cli task create --prompt "..."         # 没有 --prompt，提示词是位置参数
verge-cli task create -f a.png,b.png         # 不能逗号合并，多张写多个 -f
verge-cli task create -r 1024x1024           # 档位只有 1080p / 2k / 4k；要像素请用 -n 等张数后裁剪
```

不确定时跑 `verge-cli help <命令>`，别猜。

## 别用本 skill

- **识别 / OCR / 读图问答 / 给已有图片起标题 / 看图说话** —— 用 vision 类 skill，不要出图。
- **裁剪 / 缩放 / 格式转换等纯本地处理** —— 用 ImageMagick / Pillow，不需要 API。
- **动画 / 视频** —— 本 skill 只覆盖图像生成接口。

## 怎么告诉用户

成功 / 失败 / 部分成功时**固定话术**，agent 直接复用：

| 情况 | 怎么说 |
| --- | --- |
| 全部成功 | `已生成 {N} 张图，保存在 {DIR}/ 下：{file1}, {file2}...（链接 {天数} 天后失效）` |
| 部分成功 | `生成 {N} 张，成功 {M} 张；失败原因：{error.code} {error.message}` |
| 等待超时（出口码 9） | `本地不再等了，task_id={id}，需要继续等请说一声` |
| 任务失败（出口码 8） | `{error.code}：{error.message}（request_id={rid}）。建议改提示词再试，不要原样重跑` |
| 认证/额度/参数错（出口码 3/4/5） | **停下来告诉用户原因**，不要自己重试 |
| 拿到 task_id 但用户没要求立即等 | `已提交：task_id={id}。要等结果吗？要下载到本地吗？` |
| prepare 后上传失败 | `task_id={id} 的 submit 未执行；原任务会在上传阶段超时。需要重新运行完整任务获取新上传槽位` |
| submit 结果不明确 | `task_id={id} 的提交结果未确认；我会查询原任务，不会重复 submit` |

**关键原则**：
- 透传 `error.code` / `message` / `request_id`，不要自己编原因
- 路径给绝对路径或相对于 cwd 的清晰路径，别只甩文件名
- 链接有有效期，告知用户

## 按需加载参考（按触发）

| 触发 | 读哪份 |
| --- | --- |
| 首次使用 / 新机器 / `no API key found` | [references/setup.md](references/setup.md) |
| 看到不熟悉的 flag / 想确认模型分辨率 / 要用 `--json` 接 jq | [references/flags.md](references/flags.md) |
| 出口码 8 / 9 / 任何报错 / `unexpected response` | [references/errors.md](references/errors.md) |

本文与 CLI 冲突时以 `verge-cli help <命令>` 为准。
