<div align="center">

# verge-skill

**教 AI agent 用 [verge-cli](https://github.com/img-verge/verge-cli) 调 [Verge](https://api.verge-ai.xyz) 生图的 Agent Skill**

</div>

---

`verge-cli` 把 Verge 的图像接口包成了一个命令行工具，但 agent 不会自己知道这套 CLI 的约束 —— 提示词是位置参数、`[@名称]` 写错了参考图会静默失效、本地与 base64 图片会自动走三阶段上传、POST 不自动重试、prepare 后失败要用已有 `task_id` 恢复。这个 skill 就是把这些交给 agent。

## 能力

| 场景 | skill 教 agent 怎么做 |
| --- | --- |
| 文生图 | `verge-cli task create "提示词" --wait -o ./out`，提示词是位置参数 |
| 按参考图改图 / 合成 | `verge-cli task create -f 名称=文件`，三段式上传全自动 |
| 批量 / 长耗时 | 创建任务 + 轮询，`--wait-timeout` 可调 |
| 取回结果 | `-o DIR` 或 `verge-cli download <task_id>` |
| 出错 | 按出口码 0–9 分支，按稳定 `error.code` 判因，不 grep 文案 |

## 安装

把 `verge-image/` 目录复制到你的 AI 编程助手的 skills（或 rules / instructions）目录下即可。

```bash
cp -r verge-image /path/to/your/agent/skills/
```

常见路径参考（以几个热门工具为例，其他 agent 请查对应文档）：

| AI 编程助手 | 全局路径 |
|-------------|----------|
| Claude Code | `~/.claude/skills/` |
| opencode | `~/.config/opencode/skills/` |
| Cursor | `.cursor/rules/` |

装好后对 AI agent 说「生成图片 / 文生图 / 按参考图改图」即可命中本 skill。

## 前置条件

1. `verge-cli` 可执行文件在 `PATH` 里（`verge-cli version` 能跑）
2. 环境变量 `VERGE_API_KEY`
3. 自建部署另设 `VERGE_API_BASE_URL`

两者都没有时 skill 会引导 agent 去读 `references/setup.md`，那里有从 Docker 编译到配置 Key 的完整步骤。

## 快速开始

配好之后直接对 agent 说人话即可：

> 帮我生成一张雨夜霓虹街头的图，16:9，2k

> 把 ./char.png 里的角色放进 ./bg.jpg 的背景里，存到 ./out

agent 会自己选择立即等待还是稍后查询、命名参考图、调用 `task create`、轮询并落盘。

## 文件

```
verge-image/
├── SKILL.md                 主文档：执行模型、决策表、硬规则、不要发明的写法
└── references/
    ├── flags.md             每个命令的完整 flag、模型 × 分辨率、参考图规则、--json 字段
    ├── errors.md            出口码 0–9、任务状态机、服务端错误码、stderr 快查表
    └── setup.md             二进制从哪来、Key 怎么配、首次自检
```

`references/` 里的三份只在需要时按需加载，不会一直占着 agent 的上下文。

## 数据说明

提示词、参考图（本地文件与 base64 解码结果会直传对象存储）和生成参数都会发送到 Verge 或其自建部署。生成的图片链接由服务端签发，7 天后失效（封面图 1 天）。API Key 只发往配置的接口地址；结果图下载和参考图直传对象存储时不带任何凭证。

## 相关

- [verge-cli](https://github.com/img-verge/verge-cli) —— 本 skill 驱动的命令行工具
- [Verge](https://api.verge-ai.xyz) —— 上游服务
