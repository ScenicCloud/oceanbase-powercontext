# 运行说明与验证样例

作品：**23:57 时间循环便利店**。Dify Chatflow 编排、AI 店员即兴对白、PowerContext 保存跨轮记忆的解谜小游戏。所有提交资产位于本目录 `dify/`。

## 1. 作品与玩法

你被困在一家永远停在 23:57 的便利店。每轮有三次行动，找齐“收据”“广播”两条线索后获得额外一次行动。物品在“重来”后清零，但发现的秘密会写入 PowerContext，在后续轮次和新会话中继续生效。

- 输入“开始”进入第 1 轮；输入“状态”查看轮次、行动数、物品和记忆。
- 可用行动：看收据、听广播、买牛奶、拉电闸、递牛奶、开门、重来、状态。
- 通关路径：看收据 → 听广播 → 重来 → 买牛奶 → 拉电闸（依赖广播记忆） → 递牛奶（依赖收据记忆） → 开门（真结局）。
- 通行证（player ticket）是存档凭据；换新通行证即为全新存档。
- 暂不支持自由自然语言动作判定，请使用菜单中的行动词。

## 2. 技术分工

| 组件 | 作用 |
| --- | --- |
| Dify | 核心编排：输入 → Code 编码 → HTTP 调用游戏后端 → Code 提取权威结果 → LLM 店员对白 → Answer。共 6 个节点，正式 AI 版全部真实运行。 |
| PowerContext | 每个玩家一个独立 Scope。观察类行动通过 `/v1/memory/remember` 写入发现（收据、广播），每次行动通过 `/v1/context/prepare` 召回；电闸与递牛奶的判定实际依赖召回结果，服务不可用会显式失败。 |
| Python 后端（`game.py`） | 确定性判定行动、物品、电闸、信任和结局；SQLite 保存本轮状态与请求去重。跨轮笔记只从 PowerContext 读取。 |
| 模型 | 只负责生成氛围对白，不修改权威状态、不决定胜负。 |

不使用 seekdb、Embedding、Rerank、外部知识库，也不依赖自定义 Dify 插件；模型插件使用官方 `openai_api_compatible` 安装包（已随附）。

## 3. 交付文件

| 文件 | 用途 |
| --- | --- |
| `time-loop.yml` | 正式 AI Chatflow，Dify 1.13.0，DSL 0.6.0，6 节点 |
| `time-loop-no-model.yml` | 免模型 5 节点版，用于只检查流程、网络与记忆；与正式版无导入顺序依赖 |
| `game.py` / `start.sh` / `play.py` | 游戏服务、PowerContext 启动脚本、本地终端入口 |
| `deploy/` | Dify 1.13.0 部署、初始化与模型配置脚本，详见 `deploy/README.md` |
| `plugins/` | 官方 `openai_api_compatible` 0.0.73 安装包及来源、校验值 |
| `verify_dify.py` / `verify_published.py` / `verify.py` | Console API 导入与玩法验收、已发布应用 `/v1/chat-messages` 验收、后端补充测试 |
| `evidence/` | 导入响应、原始 SSE 事件、节点输出、发布结果、模型与运行环境记录 |

## 4. 运行环境

- Python 3.12、Git、uv、Docker（ARM64 或 AMD64），至少 6 GiB 可用内存
- Dify API/Web/Worker `1.13.0`，Sandbox `0.2.12`，Plugin Daemon `0.5.3-local`，PostgreSQL 15、Redis 6
- PowerContext 固定提交 `dab240063bfcfa7917898dc44da4baadb3747bd0`，安装 `cli,server` extras
- 模型：默认 `deepseek-flash`（`https://api.deepseek.com/v1`），可用 `MODEL_BASE_URL`、`MODEL_NAME` 替换；更换模型名时同步修改 DSL 的 LLM 节点
- 应用本体只依赖 Python 标准库；部署脚本使用环境内已有的 httpx 与 PyYAML

## 5. 启动步骤

在 `dify/` 目录执行，并保持终端运行：

```bash
GAME_HOST=0.0.0.0 bash start.sh
```

脚本会创建 `.venv`、安装固定 PowerContext 提交，并启动游戏（18766）与 PowerContext（18765）。另开终端，仍在 `dify/` 目录：

```bash
.venv/bin/python deploy/prepare_stack.py
docker-compose -f .runtime/dify-stack/compose.yml --env-file .runtime/dify-stack/.env up -d

export MODEL_API_KEY='<自己的模型 API Key>'
.venv/bin/python deploy/install_model.py
unset MODEL_API_KEY
```

- Dify 工作台：http://localhost:18771
- Dify 正式 API：http://localhost:18770
- 本地验收管理员凭据：`.runtime/console-account.json`（仅当前用户可读，不入库）

DSL 环境变量 `GAME_URL` 默认 `http://host.docker.internal:18766`，必须是 **Dify 服务端**可访问的地址。Linux Docker 需配置 host-gateway 或可达的宿主机地址；Dify Cloud 无法访问本机 localhost。

## 6. 导入 DSL（手动方式）

1. Dify 工作室 → 导入 DSL → 上传 `time-loop.yml`（也可先导入 `time-loop-no-model.yml` 做免模型联通检查）。
2. 安装 `plugins/openai_api_compatible-0.0.73.difypkg`；在 OpenAI-API-compatible 中添加聊天模型 `deepseek-flash`，填入自己的 API Key 与 `https://api.deepseek.com/v1`，chat 模式、扩展兼容模式、不支持思考输出。
3. 确认环境变量 `GAME_URL` 指向 Dify 可访问的游戏服务地址。
4. 发布 Chatflow，打开调试入口。

## 7. 验证样例

每位玩家使用不同的随机通行证，按顺序输入：

| 步骤 | 输入 | 预期表现 |
| --- | --- | --- |
| 1 | 新通行证，`开始` | 第 1 轮，笔记为空 |
| 2 | `看收据` | 收据秘密写入 PowerContext，笔记出现 1 条 |
| 3 | `听广播` | 广播秘密写入，笔记 2 条，获得额外行动 |
| 4 | `重来` | 第 2 轮，物品清空，笔记保留 |
| 5 | 新建 Dify 会话，同一通行证，`状态` | 新 conversation_id 中仍能读到两条记忆 |
| 6 | `买牛奶` | 得到热牛奶 |
| 7 | `拉电闸` | 依赖广播记忆成功断电 |
| 8 | `递牛奶` | 收据记忆 + 牛奶 + 断电共同产生店员信任 |
| 9 | `开门` | 真结局“明天照常营业” |

另一通行证直接 `开门` 应进入坏结局并重置本轮，用于验证玩家隔离与存档恢复。

## 8. 自动化验收与证据

```bash
.venv/bin/python verify_dify.py        # 正式 Console API：导入、节点、完整玩法、跨会话记忆、玩家隔离
.venv/bin/python verify_published.py   # 已发布应用 /v1/chat-messages
.venv/bin/python verify.py             # 游戏与 PowerContext 补充测试
```

证据在 `evidence/dify/`：`00-no-model.json` 为免模型 5 节点成功记录，`01-ai.json` 至 `11-bad-ending.json` 覆盖 AI 流程、两个会话、两个玩家与结局，`production-*.json` 为发布后正式 API 记录，`imports.json` 记录 DSL 的 SHA256，`runtime.json` / `plugin-runtime.json` 记录实际镜像与插件来源。

## 9. 已知限制

- 游戏服务为单进程演示设计，不支持多实例并发。
- 客户端需复用同一 request_id 才能对网络重试去重；Dify HTTP 节点重试复用同一次 workflow_run_id。
- 跨系统无分布式事务：故障时线索可能先写入而状态未提交，重试会检查已存在标记。模型调用失败时行动可能已提交，请先用“状态”核对。
- 部署服务与模型 API Key 均由运行者提供。

## 10. 来源

基于官方 `oceanbase/powercontext` 的 `2026-competition-ecosystem` 分支开发；Dify 部署源文件来自 Dify 1.13.0 官方 Docker 目录，模型插件为官方 marketplace 安装包。
