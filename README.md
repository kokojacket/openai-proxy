# bps-proxy

> **当前状态：此方法已经失效。BPS 渠道已开始降智，模型质量和工具调用稳定性不再达到本文档所述效果；本文档内容仅作历史记录。**

把本机 Codex 等客户端的 Responses 请求转到 Excel 插件后端。客户端自己的工具会经 `run_officejs` 转接，仍由客户端执行。

社区交流：[LINUX DO](https://linux.do/)。

这是独立的社区项目，与 OpenAI 没有隶属或背书关系。上游接口及账号可用性可能变化。

默认监听本机 `127.0.0.1`，也支持 `localhost` 和 IPv6 回环地址 `::1`；不允许绑定 `0.0.0.0` 等对外地址，也不接受浏览器网页直接请求。代理使用你的登录态访问上游，请只在可信的个人电脑上运行。

## 效果参考

使用本代理后，由 GPT-6 Astra 生成的 HTML / SVG 动画示例：鹈鹕驾驶跑车的海岸场景。

![GPT-6 Astra 生成的鹈鹕跑车 SVG 动画效果](docs/images/pelican-svg-example.png)

## 需要

- Python 3.9 或更高；安装时会一并安装 zstandard，用于读取 Codex 的压缩请求
- 本机 Codex 已经登录。默认读取 `~/.codex/auth.json`；设置了 `CODEX_HOME` 时读取该目录下的 `auth.json`

## 安装

```bash
git clone https://github.com/kokojacket/openai-proxy.git
cd openai-proxy
```

在 Python 虚拟环境中安装依赖和命令行入口：

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install .
bps-proxy --help
```

Windows 上使用 `python` 代替 `python3`，通过 `.venv\Scripts\activate` 激活虚拟环境。

## 启动

macOS / Linux：

```bash
./start.sh
```

Windows：

```bat
start.bat
```

也可以：

```bash
python3 -m bps_proxy
```

默认地址是 `http://127.0.0.1:8787/v1`。换端口：`python3 -m bps_proxy --port 8788`。

确认服务已启动：

```bash
curl http://127.0.0.1:8787/health
```

健康接口仅检查本机代理是否可访问，不检查登录态或上游可用性。日志输出到终端。端口被占用时，停止占用它的旧进程或换一个端口。

### 并发与限速

这些限制在同一个代理进程内共享，按模型请求计数，不按聊天窗口计数。一个会话中的子任务或后台工作也可能产生重叠请求。

| 参数 | 默认值 | 含义 |
| --- | --- | --- |
| `--max-concurrent` | 8 | 同时处理的模型请求数 |
| `--max-pending` | 32 | 活动请求已满时的等待队列容量 |
| `--queue-timeout` | 120 | 等待进入处理阶段的最长秒数 |
| `--upstream-rps` | 5 | 滚动 1 秒窗口内最多发起的上游请求数 |

超出活动请求上限后按到达顺序排队；只有队列已满或等待超时才返回 503，并带 `Retry-After: 1`。健康检查和模型目录查询不占模型请求名额。

上游发起限速与活动请求上限分别生效。首次生成、每次重试和实际附件上传共用每秒额度，超出后等待；附件缓存命中不计数。排队的 120 秒上限不代表整个生成过程的超时。

例如，显式指定默认参数：

```bash
./start.sh --max-concurrent 8 --max-pending 32 --queue-timeout 120 --upstream-rps 5
```

这些是代理的默认保护参数，参考 ghcp_proxy 的连接池与请求发起策略，不代表 OpenAI 公布的账号配额。

### 流式响应大小

上游 SSE 的单行和单事件默认上限均为 **16 MiB**，可用正整数参数调整，例如：

```bash
./start.sh --max-sse-event-mib 32
```

这是响应字节大小限制，与模型的上下文 token 上限分别生效。调高会增加并发请求可能占用的内存；修改启动参数后需重启代理。

启动日志的 `max_sse_event_bytes` 显示实际生效值。超限日志记录 `request_id`、事件类型、行或事件限制、已读取字节数及上限，不记录响应正文。单行超限时只读取到上限加 1 字节，因此日志中的 `observed_bytes` 是已观测值，不是完整行大小。至少 4 MiB 的成功事件另记 `event_bytes`，便于判断是否接近上限。

## 接到 Codex

在用户配置 `~/.codex/config.toml` 顶层加一行，放在所有 `[table]` 之前；设置了 `CODEX_HOME` 时修改对应目录的 `config.toml`：

```toml
openai_base_url = "http://127.0.0.1:8787/v1"
```

这使用 Codex 内置的 `openai` provider，不需要另建 provider。原有模型和推理档位设置可以保留；实际可用模型见下方目录。

从旧配置迁移时，删除 `model_provider = "bps"` 和 `[model_providers.bps]` 段。如果设置了其他自定义 `model_provider`，也需删掉该选择或改为 `"openai"`。旧的自定义 provider 接法仍然兼容。

保存后重启 Codex 客户端并新建对话，使连接地址与模型目录重新加载。该地址应放在用户配置中，项目内的 `.codex/config.toml` 不适合配置连接地址。代理没启动时，Codex 会连不上。

要回到官方通道，删掉这一行 `openai_base_url`，再重启 Codex。

### 确认请求经过代理

先检查本机服务与目录：

```bash
curl --fail http://127.0.0.1:8787/health
curl --fail http://127.0.0.1:8787/v1/models
```

然后在 Codex 新建对话并发出一条请求，按相同的 `request_id` 对照代理日志：

- `service configuration` 显示已加载代码的 `build` 摘要、并发参数、SSE 大小上限、目录版本与校验值。
- `local request` 应出现 `POST /v1/responses`；`upstream start` 分别记录 `requested_model` 和实际 `model`，可用于核对型号映射。
- `relay ended` 的 `terminal=response.completed` 表示该次响应已完成。HTTP 200 或健康接口成功本身不能证明模型请求完成；流式响应仍可能以 `response.failed` 结束。

Responses Lite 请求的工具来源记为 `additional_tools`，普通请求记为 `top_level`。工具数量取决于客户端本次声明，不应仅凭顶层 `tools` 缺失判断工具没有转发。WebSocket 升级后的 426 用于回退到 HTTP/SSE；带 `Origin` 的浏览器请求会被本机守卫拒绝。

### 更新已有安装

拉取新代码后，在虚拟环境中重新安装依赖和命令行入口：

```bash
git pull --ff-only
python -m pip install .
```

随后使用原来的启动方式重启代理，再重启 Codex。已经运行的 Python 进程不会自动加载磁盘上的新代码；仅重启 Codex 也不会更新代理进程。

## 并行验证新版本

macOS / Linux 上，可以在独立工作副本中启动候选服务：

```bash
python3 tools/candidate.py start
python3 tools/candidate.py status
python3 tools/candidate.py run
python3 tools/candidate.py stop
```

默认候选端口为 `18787`，缓存与日志位于 `~/.bps-proxy/candidates/<副本标识>/`。`run` 通过 Codex 的 `-c` 参数指定本次连接地址，不写入共享的 `~/.codex/config.toml`。ChatGPT 桌面端与独立 CLI 的现有连接配置不会因此改变。

若端口已被占用，使用 `python3 tools/candidate.py --port 18788 start` 和相同端口的 `run`。工具不会停止占用端口的其他服务，也不允许用候选模式绑定 `8787`。`stop` 只停止本副本启动的候选进程。

验证图片可以使用：

```bash
python3 tools/candidate.py run -- exec --image /path/to/image.png '描述这张图片'
```

完成验证后再安排桌面端切换；已有对话依赖原代理时，保持原进程和共享配置。候选缓存独立，切换后应开启新对话，不应假定能接续原代理缓存中的工具调用。

## 图片

支持用户消息中的图片，以及工具结果数组中的截图。PNG、JPEG、GIF、WebP 会检查编码、格式头部、尺寸及大小。单张最多 20 MiB，图片解码后合计最多 32 MiB；整个 JSON 请求仍受 32 MiB 限制，因此 base64 的编码开销也会计入请求体。

内联图片被上游拒绝时，会尝试上传到 BPS 附件接口并使用返回的附件编号。同账号的重复图片复用缓存，附件失效时重新上传；不需要自行搭建公网图片服务。图片会发送至 OpenAI 的 BPS 服务，代理不能控制该服务的数据保留期限。

图片格式错误、上传失败或附件仍被拒绝时会返回错误，不会省略图片后继续回答。工具结果中的相邻文字和图片数组会一并保留。

## 模型和档位

代理的模型目录列出：`gpt-6-astra`、`gpt-5.6-sol`、`gpt-5.6-luna`、`gpt-5.6-terra`。实际可用性取决于上游和账号权限。

模型指令和元数据来自随包发布的 Codex 0.155.0 目录快照，保留四个模型的完整模板、`tokens / 10000` 工具输出截断策略和 `use_responses_lite: true`。客户端自定义的 `model_instructions_file` 仍由 Codex 决定优先级。该快照的版本与校验值可在启动日志查看。

这里的 Responses Lite 是请求协议设置：工具目录可放在 `input` 中的 developer `additional_tools` 项里。它与模型名称、推理档位是不同的设置；代理同时兼容 Lite 和顶层 `tools` 两种声明方式。

需要使用另一份目录时，可通过 `--model-catalog /path/to/catalog.json` 或 `BPS_MODEL_CATALOG` 指定；目录必须包含四个型号及完整元数据。启动后固定使用这份快照，更新文件后需重启；无效目录会报告错误。开发者可以使用 `tools/import_model_catalog.py SOURCE --output NEW_FILE` 从明确选择的目录文件生成去除账号身份字段的快照。

以下请求名称固定映射到对应的 5.6 型号；Codex 主会话、子任务及后台工作发出的普通 Responses 请求都使用同一映射：

| 请求型号 | 实际转发型号 |
| --- | --- |
| gpt-6-sol | gpt-5.6-sol |
| gpt-6-terra | gpt-5.6-terra |
| gpt-6-luna | gpt-5.6-luna |

`gpt-6-astra` 和原有 5.6 型号保持原样，`gpt-5.6-terra` 不会转到 Luna。以上是固定别名映射，不是请求失败后的降级。日志分别记录请求型号与实际型号；上游拒绝或失败时，不再改用其他模型重试。

`low`、`medium`、`high`、`xhigh`、`ultra` 原样送出。这个后端没有 `max`，选这一档时代理会改成 `xhigh`。

## 请求与工具

- 提供 `POST /v1/responses`、`POST /v1/responses/compact`、`GET /v1/models` 和 `GET /health`，同时兼容省略 `/v1` 的路径。
- 上下文压缩使用 BPS 原生压缩，返回的不透明历史可随下一次请求继续发送。上游必须返回恰好一条有效压缩项；数量或流式事件不一致、失败、断流、仅返回普通文字时都会报告失败。
- WebSocket 升级请求返回 `426`，供 Codex 回退到 HTTP/SSE。
- `input` 使用字符串或消息数组；`stream` 使用布尔值。支持流式和非流式响应，也会保留上游的 `incomplete`、`failed` 状态。
- 同时识别顶层 `tools` 和 Responses Lite 的 developer `additional_tools` 声明，校验后统一转为工具隧道目录。
- 支持普通函数工具及 `custom` 文本工具。参数类型、必填项、枚举和嵌套约束会转成自然语言工具目录，不向上游发送客户端 `tools` 或 `tool_choice` 字段。工具结果保留文本和图片内容；`tool_choice: "none"` 会禁用客户端工具，未声明的工具不会被自动启用。格式有歧义的可执行脚本会要求重新生成，不猜测补写引号。
- `tool_choice: "required"` 或指定工具时，必须返回有效的客户端调用；没有匹配工具返回 `400`，漏调用经一次纠正仍未恢复则返回 `response.failed`。上游的拒绝回答、`failed` 和 `incomplete` 状态保持原样。已有会话省略工具声明或用顶层空数组续接时沿用原目录；Lite 的 `additional_tools` 是完整声明，显式空目录会撤销此前工具。明确禁用本次工具请用 `tool_choice: "none"`。
- 每次请求需携带完整对话历史。不支持 `previous_response_id`、`conversation` 或 `background: true` 异步响应模式；这类请求会返回 `400`。
- 支持未压缩 JSON 以及 `Content-Encoding: zstd`、`gzip`、`deflate`。接收的请求体及解压后的内容均最多 32 MiB；损坏、截断或拼接的压缩数据会被拒绝。
- 请求需要 `Content-Length`，不接受分块上传；读取客户端请求体超时为 30 秒。这是 Codex 模型请求的兼容接口，不包含登录、账户管理或其他 OpenAI 产品接口。

## 登录与缓存

登录态无效或过期时返回 `401`；请重新登录本机 Codex。代理不会自动刷新登录令牌。

新调用按账号和会话隔离。工具回放缓存默认位于 `~/.bps-proxy/calls.json`，可以通过 `--state /path/to/calls.json` 更换位置。缓存包含完整的原始调用（包括 `id`、`summary` 和 `references`）与轮次计数，请不要上传或分享；在 macOS / Linux 上，新写入的缓存仅允许当前用户读写。调用与轮次计数各最多保留 512 条，写盘失败时会记录警告并继续使用内存中的记录。缓存丢失或被淘汰后，重建的调用无法保证与原始调用完全一致。

同一轮工具续接保持 `turn_id`，内部重试会递增 `agent_iteration`；客户端重复提交同一阶段保持计数，收到下一阶段的工具结果后继续递增。计数会随缓存保存，避免内部重试后收到客户端工具结果时回退。旧格式缓存可读取，但无作用域的旧条目不会自动用于新账号或新会话。

普通文本增量转发。可执行工具调用只在上游正式完成响应后交付；断流、事件损坏或连续工具转接失败会保留失败状态，不合成成功响应。网络超时返回 `504`，其他连接错误返回 `502`；如果流式响应已经开始，则通过 SSE `response.failed` 事件报告。上游 SSE 单行及单事件默认最多 16 MiB，可通过 `--max-sse-event-mib` 调整；超限会关闭上游连接并报告失败。空闲连接超时为 5 分钟，单次上游流最长为 15 分钟。

Excel 行为指令由后端注入，代理无法将其从模型实际收到的提示中删除。返回给客户端的 `instructions` 字段会还原为客户端原文，但这不代表后端提示词已被清除；该转接方式无法保证与官方 Responses API 完全一致。

## 开发验证

```bash
python3 -m unittest discover -s tests -v
```

测试使用临时文件、回环 HTTP 服务和模拟上游，不需要真实账号或外部请求。

安装了 Codex CLI 后，还可以检查真实客户端与代理的兼容性：

```bash
python3 tools/probe_compatibility.py
```

该脚本使用临时配置、模拟凭据和模拟模型响应，验证四个模型的 Lite 工具调用、真实临时文件写入与回读、完整调用回放、模型目录、请求解压、WebSocket 回退和压缩后续聊。它不修改现有 Codex 配置，也不验证真实 BPS 账号或上游服务。

协议细节与测试覆盖见 [Codex 兼容说明](docs/codex-compatibility.md)。

## 实现参考

SSE 单行与聚合事件的 16 MiB 保护参考 ranxi2001/sub2api 的 BPS 读取器（核对版本 `fe27f98`，`backend/internal/service/basispoints/stream.go`）；本代理增加了独立的启动参数和脱敏大小诊断。

图片附件上传与重传流程参考 Kaixxrua/excel-codex-bridge（核对版本 `66c41df`），请求解压边界参考其 `8a277df` 版本。请求限额、图片隔离与终态校验参考 ranxi2001/sub2api 的 BPS 通道（核对版本 `d215edd`），原生压缩触发方式参考其 `00bdb50` 版本。采用独立实现，保留轻量 HTTP 服务，仅增加 zstandard 依赖；没有引入这些项目的服务端框架或公网图片中转。

并发与发起频率参考 Nonary/ghcp_proxy（核对版本 `dfb758b`）：其 Excel HTTP/1.1 连接池最多 8 条连接，上游发起按每秒 5 次限速。本代理采用 8 个活动模型请求、有界 FIFO 排队和共享滚动窗口实现，重试与实际上传同样计入额度；没有引入 HTTPX 连接池。

## 贡献

欢迎通过 GitHub Issues 报告问题，或提交 Pull Request。问题报告请附上 Python 版本、操作系统、复现步骤和已脱敏的错误信息。请勿上传登录令牌、`auth.json`、对话内容或工具调用缓存。

## 许可证

本项目采用 [MIT License](LICENSE)。
