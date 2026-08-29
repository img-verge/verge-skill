# 安装与配置

本文假定**用户已安装 `verge-cli`**。agent 的工作是：检测 / 配 Key / 自检 / 诊断，**不负责安装或编译**。

二进制装不上、找不到对应平台 → 让用户去 [verge-cli Releases](https://github.com/img-verge/verge-cli/releases) 下，不在本 skill 范围内。

## 检测

```bash
verge-cli version    # 装了显示 v0.1.0；源码编的会显示 dev（不影响用）
```

找不到时报清楚，**别自己尝试编译**：

> `verge-cli` 未安装。请从 [Releases](https://github.com/img-verge/verge-cli/releases) 下载对应平台版本放进 `PATH`，或 `go install github.com/img-verge/verge-cli@latest`。装好后再说一声。

## 配置 Key

优先级：**`--api-key` flag > `$VERGE_API_KEY` > 配置文件**。

agent 优先用环境变量：

```bash
export VERGE_API_KEY=sk-...
```

**不要把 Key 写在命令行里**（`verge-cli --api-key sk-... ...`）—— 会进 shell 历史和日志。要长期保存就写配置文件：

```bash
verge-cli config set api-key sk-...       # 文件权限收到 0600
verge-cli config show                     # Key 已掩码，可放心贴出
verge-cli config path                     # 配置文件实际位置
```

配置文件位置：Windows `%APPDATA%\verge\config.json`；macOS / Linux `$XDG_CONFIG_HOME/verge/config.json`，未设则 `~/.config/verge/config.json`。`VERGE_CONFIG` 可整体覆盖路径。

自建部署另设接口地址（缺 `/v1` 会自动补上）：

```bash
export VERGE_API_BASE_URL=https://gateway.example.com
# 或 verge-cli config set base-url https://gateway.example.com
```

也可以顺手设默认出图参数，之后每次生成都不用再带：

```bash
verge-cli config set model gpt-image-2
verge-cli config set resolution 2k
verge-cli config set aspect-ratio 16:9
```

## 自检

```bash
verge-cli balance
```

一条命令同时验证 Key 与网络：**只读、不创建任务、不扣额度**。输出形如

```
wallet available  1,234,567
key available     unlimited
```

`key available` 显示 `unlimited` 表示这个 Key 没有单独额度上限；显示 `0` 表示有上限且已用尽 —— 两者含义完全不同。

## 常见失败

| 现象 | 原因与修法 |
| --- | --- |
| `command not found: verge-cli` | 二进制不在 PATH。**不要自己装**；让用户按上面「检测」节装 |
| `no API key found` | 按上面配 Key。stderr 会把三种配法都列出来 |
| 出口码 3（`invalid_api_key` / `token_expired` / `token_disabled`） | Key 无效、过期或被禁用，告诉用户去 Verge 后台换一个 |
| 出口码 4（`insufficient_quota`） | 跑 `verge-cli balance` 看具体余额，告诉用户充值 |
| 请求打到了错误的域名 | 自建部署没设 `VERGE_API_BASE_URL`；加 `--verbose` 能看到每个请求的实际地址 |