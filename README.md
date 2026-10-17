<div align="center">

# verge-skill

**两个给 AI agent 用的出图 skill：用 [verge-cli](https://github.com/img-verge/verge-cli) 调 [Verge API](https://api.verge-ai.xyz/v1) 生图 + GPT-Image2 提示词风格库**

</div>

---

出图这件事 agent 有两道坎：**提示词写得不像样**，和 **CLI 约束不知道**。这个仓库放了两个 skill，各管一道：

- `gpt-image-2-style-library/` —— **写提示词**：22 个模板、13 类目、19 风格、10 场景，中英双语关键词匹配，把用户一句大白话变成生产级提示词。零依赖，装了就能用。
- `verge-image/` —— **执行出图**：`verge-cli` 把 Verge 的图像接口包成了一个命令行工具，但 agent 不会自己知道这套 CLI 的约束 —— 提示词是位置参数、`[@名称]` 写错了参考图会静默失效、本地与 base64 图片会自动走三阶段上传、POST 不自动重试、prepare 后失败要用已有 `task_id` 恢复。这个 skill 就是把这些交给 agent。

两个可以配合：先让风格库出提示词，再让 `verge-image` 执行。

## 两个 skill

| skill | 干什么 | 依赖 |
| --- | --- | --- |
| `gpt-image-2-style-library/` | 挑风格、写生产级提示词（22 模板 / 13 类目 / 19 风格 / 10 场景） | 无 |
| `verge-image/` | 驱动 `verge-cli` 生成 / 改图 / 批量 / 取回结果 / 出错处理 | `verge-cli` + `VERGE_API_KEY` |

## verge-image 能力

| 场景 | skill 教 agent 怎么做 |
| --- | --- |
| 文生图 | `verge-cli task create "提示词" --wait -o ./out`，提示词是位置参数 |
| 按参考图改图 / 合成 | `verge-cli task create -f 名称=文件`，三段式上传全自动 |
| 批量 / 长耗时 | 创建任务 + 轮询，`--wait-timeout` 可调 |
| 取回结果 | `-o DIR` 或 `verge-cli download <task_id>` |
| 出错 | 按出口码 0–9 分支，按稳定 `error.code` 判因，不 grep 文案 |

## 安装

把要装的 skill 目录复制到你的 AI 编程助手的 skills（或 rules / instructions）目录下即可：

```bash
# 只装一个
cp -r verge-image /path/to/your/agent/skills/
cp -r gpt-image-2-style-library /path/to/your/agent/skills/

# 两个都装
cp -r verge-image gpt-image-2-style-library /path/to/your/agent/skills/
```

常见路径参考（以几个热门工具为例，其他 agent 请查对应文档）：

| AI 编程助手 | 全局路径 |
|-------------|----------|
| Claude Code | `~/.claude/skills/` |
| opencode | `~/.config/opencode/skills/` |
| Cursor | `.cursor/rules/` |

装好后：

- 说「生成图片 / 文生图 / 按参考图改图」→ 命中 `verge-image`
- 说「给这个需求挑模板写提示词 / 用某某风格出图」→ 命中 `gpt-image-2-style-library`

## 前置条件

**`gpt-image-2-style-library`**：无。不需要 `verge-cli`，不需要 API Key。

**`verge-image`** 需要：

1. `verge-cli` 可执行文件在 `PATH` 里（`verge-cli version` 能跑）
2. 环境变量 `VERGE_API_KEY`
3. 自建部署另设 `VERGE_API_BASE_URL`

任一项异常时 skill 会引导 agent 读 `verge-image/references/setup.md`，那里有检测、配 Key、自检和常见失败对照。

## 快速开始

配好之后直接对 agent 说人话即可：

> 帮我生成一张雨夜霓虹街头的图，16:9，2k

> 把 ./char.png 里的角色放进 ./bg.jpg 的背景里，存到 ./out

> 给「咖啡店新品促销海报」用风格库挑个模板，写一版提示词

agent 会自己选择立即等待还是稍后查询、命名参考图、调用 `task create`、轮询并落盘；风格库那条会先按类目 / 风格 / 场景匹配，给出选中的模板名和可直接复制的提示词。

## 文件

```
verge-image/
├── SKILL.md                 主文档：预检、执行模型、决策表、硬规则、不要发明的写法
└── references/
    ├── flags.md             每个命令的完整 flag、模型 × 分辨率、参考图规则、--json 字段
    ├── errors.md            出口码 0–9、任务状态机、服务端错误码、stderr 快查表
    └── setup.md             检测、配 Key、自检、常见失败对照

gpt-image-2-style-library/
├── SKILL.md                 主文档：选模板的工作流、输出默认值
├── references/
│   └── style-library.md     22 个模板全量索引：类目、风格、场景、Guidance、Pitfalls
├── assets/                  示例输出图
└── package.json / bin/ / agents/   上游 npm 发布用，skill 运行不需要
```

`references/` 里的内容只在需要时按需加载，不会一直占着 agent 的上下文。

## 数据说明

仅 `verge-image` 涉及数据外发：提示词、参考图（本地文件与 base64 解码结果会直传对象存储）和生成参数都会发送到 Verge 或其自建部署。生成的图片链接由服务端签发，7 天后失效（封面图 1 天）。API Key 只发往配置的接口地址；结果图下载和参考图直传对象存储时不带任何凭证。

`gpt-image-2-style-library` 不依赖任何外部服务，模板与规则全在本地 `references/` 里。

## 相关

- [verge-cli](https://github.com/img-verge/verge-cli) —— `verge-image` 驱动的命令行工具
- [Verge](https://verge-ai.xyz) —— 上游服务
- [awesome-gpt-image-2](https://github.com/freestylefly/awesome-gpt-image-2) —— `gpt-image-2-style-library` 的模板来源，MIT License，本仓库原样收录其 skill 产物
