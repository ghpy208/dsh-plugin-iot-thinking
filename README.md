# dsh-plugin-iot-thinking

为 **DeepSeek Harness 0.2.x** 提供的 IoT thinking 块压缩插件，以**外部 bundle** 形式分发
（可在 设置 → 添加插件 里填绝对路径安装，也可以用 `dsh plugin add`）。

> **Fork 说明。** 本仓库是
> [`Roninton/IoT-items-of-thoughts`](https://github.com/Roninton/IoT-items-of-thoughts)
> 中 `dsh-plugin-iot-thinking/` 的修复版 fork。要在 DSH **桌面版（Electron）** 上、
> 配合 **WorkBuddy 路由**的模型运行，需要三处修复；上游原版适用于 CLI / npm 布局。
> 详见 [本 fork 的修复](#本-fork-的修复)。

## 它解决的问题

上一代 `iot-compaction` 后端走的是 `ctx.compaction`。dsh 的 compaction 会替换**一段历史**，
并追加一条 checkpoint 消息。模型把那条 checkpoint 当成新的用户回合：

> 上下文已压缩 · 已压缩 10 条历史记录（约 5002 tokens）

然后它开始想：

> The conversation has been compacted. I need to figure out where we are…

这不是 IoT 的规则。IoT 的规则是：**thinking 块结束的那一刻，压缩这一块，并把压缩结果原地返回。**
正文、tool calls、usage 与 finish reason 保持原生行为。

## 本插件的做法

dsh 把 `llm/stream` 暴露为包住每次模型调用的 **waterfall**。适配器吐出原始 chunk，
dsh 用 waterfall 吐出的内容组装最终提交的 assistant 消息。本插件：

1. 放行 `block-start`（thinking 块正常开始）；
2. 按块序号扣住 `reasoning-delta` chunk；
3. 在 reasoning 块的 `block-end`，拿完整草稿调用 IoT 压缩器；
4. 吐回**一个紧凑的 `reasoning-delta`** 和一个**改写过的 `block-end`**；
5. 文本增量、工具调用增量、usage 与 finish 一律原样透传。

因为替换发生在 assistant 消息**提交之前**，所以没有第二条消息、没有 checkpoint、
没有 `compaction/start` / `compaction/end`，也不注册 `compaction` 服务。它也照样在
preset 会话里生效，因为钩子挂在 LLM 层，而不是 preset 内部的 compaction realm。

如果压缩失败、被关闭，或者结果为空 / 不比原草稿短，原始增量与块原样放行。

## 节点格式

压缩结果不是散文，也不是只有“正/负”两类，而是一组带标签的思维节点：

```text
[支持] ...
[反对] ...
[中性] ...
[计划] ...
[权衡] ...
[待定] ...
[风险] ...
[决定] ...
```

- 每个节点一行；**节点之间空一行**；
- 行首直接写 `[标签]`，不要 `-` / `·` / 编号；
- 目标 4~6 条，最多 8 条（`maxNodes`，可在设置里改）；
- `[中性]` 最多 2 条（`maxNeutral`），多余的会合并；
- 每条尽量 ≤80 字；不要把“我读了 X / 我运行了 Y”写进节点；
- 超量节点按优先级保留：决定 > 反对 > 权衡 > 风险 > 待定 > 计划 > 支持 > 中性；
- 没有内容的类别直接省略，不硬凑；
- 旧标签（正节点 / 否掉 / 待定 / 风险）由 `formatNodes()` 自动映射到新标签；
- 没有标签的续行会并入上一个节点；整段都没有标签时补 `[中性]`。

## 退化保护（压缩器自己陷入重复）

长会话里模型可能把上一轮的 `[中性]` 当成输出模板，压缩器也可能在 `[中性]` 后吐出
`好` / `现在` / `写` / `结束` 这种单词节点。插件在四个点拦截：

1. **输入净化** — 压缩前剥掉草稿里的 `[标签]`，删掉单词填充行，折叠连续重复；
2. **提示词** — 明确“不要续写/模仿草稿里的标签和重复词；每个节点是完整判断，至少
   12 个汉字；没内容就输出空”；
3. **低温度** — 压缩器 `temperature: 0.2`；检测到退化后用 `0.1` + 严格模式重试一次；
4. **质量门** — 单词 / 重复内容 / 自我循环评论 / “好；现在；写”合并链都判为退化；
   重试仍退化时用确定性兜底（一条有效事实 + 一条计划），绝不把循环写回会话。

对应事件：

```text
thinking-degenerate { stage: "input" | "output", action: "retry" | "fallback" | "skip" }
```

如果看到 `fallback: true`，说明这次压缩器输出被判退化，会话里写回的是兜底节点；
原草稿仍在 `originals/`。实测：一段曾经产出 9 条 `[中性]` 单词节点的 622 字草稿，
现在稳定输出 5 条有效节点（322 字），无需重试；普通 web 会话 1161 字 → 153 字 / 3 条节点。

## 安装

### 从界面安装

设置 → 插件 → **添加插件** → 填入插件的绝对路径，例如：

```text
C:\Users\<你>\.dsh\local-plugins\dsh-plugin-iot-thinking
```

包内声明了 `dsh.bundle.patch`，所以插件管理器会把它加进 profile 的 `dsh.profile.bundles`，
它的 `cordis.patch.yml` 那一行会自动加载。

### 从命令行安装

```powershell
dsh plugin --profile desktop add "C:\Users\<你>\.dsh\local-plugins\dsh-plugin-iot-thinking"
```

> 注意：`--profile` 要填你实际在用的 profile 名。本机 GUI 跑的是 `desktop`；
> 若你没有该 profile，`dsh plugin` 会先初始化一个。

装完**重启一次** dsh（host 侧是新模块）。client 半个从 `exports["./client"]` 出。

## 开关与参数

设置 → 插件 → **IoT 思考压缩**：

- **已启用 / 已关闭** — 实时开关；
- **最小压缩字符数** — 短于此值的草稿不处理（默认 400）；
- **最大输入字符数** — 长于此值的草稿不处理（默认 24000）；
- **节点输出上限** — 压缩器的 `maxTokens`（默认 2000）；
- **最多节点数** — 节点链硬上限（默认 8）；
- **最多 [中性] 节点** — 多余的 `[中性]` 会被合并（默认 2；填 `0` 则全部删除）；
- 可选 **provider / model** — 留空表示跟随当前请求的路由，也可以指定 DSH 里注册的
  其他 provider/model。

### 压缩档位（3 / 5 / 8 标签）

设置页的 **压缩档位** 决定标签集合与硬节点上限；档位是预设，`maxNodes` /
`maxNeutral` 仍可手动覆盖，但不会超过档位硬上限。

| 档位 | 标签 | 目标 / 上限 | 说明 |
|---|---|---|---|
| `compact` | `[支持] [反对] [待定]` | 3~4 / 5 | 最接近旧 IoT 的 ~85% 压缩；8 标签会硬归并到这三类 |
| `balanced` | `[支持] [反对] [待定] [计划] [决定]` | 4~6 / 6 | 保留决定与下一步；`[中性]`→`[支持]`、`[权衡]`→`[决定]`、`[风险]`→`[反对]` |
| `full` | 8 标签（当前默认） | 4~6 / 8 | 质量优先；`[中性]` 最多 2 条 |

归并是**有损**的：`compact` 把 `[计划]` 并进 `[待定]`、`[决定]`/`[权衡]`/`[中性]`
并进 `[支持]`、`[风险]` 并进 `[反对]`。要保留“决定了什么 / 下一步”就用 `balanced`
或 `full`；要极限省 token 才用 `compact`。`thinking-compressed` 事件会记录 `profile`，
便于按档位统计 `charsIn/charsOut`。

开关写入：

```text
$DSH_HOME/plugin-data/iot-thinking/config.json
```

流钩子每次请求都会按 mtime 读这个文件，所以开关在**下一个模型请求**就生效，不用重启。

## 怎么验证它真的生效了

**只有插件账本是可靠证据**（会话日志里看见的紧凑块，分不清是插件压的还是模型自己写短的）：

```text
$DSH_HOME/plugin-data/iot-thinking/events.jsonl
```

事件：`constructed`、`block-replaced`、`block-kept`、`block-skipped`、`block-failed`、
`thinking-compressed`、`thinking-failed`、`thinking-degenerate`。

替换前的原文会在替换前归档：

```text
$DSH_HOME/plugin-data/iot-thinking/originals/<sessionId>/<time>-block<index>.txt
```

dsh 存的是 waterfall 吐出来的那些 chunk，所以块一旦被替换，原文在会话里就没了，
只存在于这个归档目录。

## 文件表

| 文件 | 作用 |
|---|---|
| `index.mjs` | host 插件：`llm/stream` 钩子 + 配置 API |
| `lib/stream.mjs` | 纯 chunk 转换器（扣住 → 压缩 → 单个 delta + block-end） |
| `lib/compress.mjs` | 辅助 `ctx.llm.stream()` 调用、低温度重试、质量门、确定性兜底 |
| `lib/think.mjs` | IoT 指令、提示词包装、草稿净化、退化检测、输出清理 |
| `lib/config-store.mjs` | 实时 JSON 配置（行内默认值 + 用户文件） |
| `lib/config-api.mjs` | 设置卡片用的精确 HTTP 路由 |
| `lib/ledger.mjs` | 事件账本 + 原文归档 |
| `lib/client.js` | 设置 → 插件 → IoT 思考压缩 卡片 |
| `selftest.mjs` | 纯转换器测试（不启动 dsh、不连模型） |

## 自测

```powershell
node selftest.mjs
```

覆盖：完成块被替换、关闭时透传、非 agent-loop 请求透传、`minChars`、压缩器失败、
空/更长输出兜底、多块、流中断、只有 block-end 没有增量、输出清理、退化输入/输出检测、
严格重试、确定性兜底。

## 已知边界

1. **块结束时的延迟。** 草稿缓冲期间 thinking 块显示为“思考中”，随后紧凑文本才出现。
   “边压边回”是明确排除的次要功能（压缩后返回）。
2. **每压一块多一次模型调用。** 压缩器默认走当前路由；需要的话可以在卡片里指定更便宜的
   provider/model。
3. **不是安全边界。** 插件以 host 权限运行，能读模型输出；只装可信来源。
4. **原文不在会话日志里。** 它归档在 `plugin-data/iot-thinking/originals/`；需要审计就保留
   这个目录。
5. **退化重试多花一次压缩调用。** 只在质量门拒绝第一次输出时发生；确定性兜底本身是本地计算。
6. **兜底是有损的。** 两次尝试都退化时，最多保留一条事实行加一条计划行；完整草稿仍在
   `originals/` 里。

## 本 fork 的修复

三处问题都是在 **DSH 桌面版（Electron）** 上配合 **WorkBuddy 路由**的模型跑出来的。
每一处都是“静默但安全地失败”：原始 thinking 块原样放行，所以插件看起来装好了，
实际一个块都没压。

### 1. `lib/seams.mjs` — 从桌面版安装目录解析宿主包

桌面版把宿主的 `@deepseek-ai/*` 包**打在 `resources/app.asar` 里面**，既不在
`$DSH_HOME/profiles/node_modules`，也不在全局 npm 根目录。原来的
`defaultNodeModulesRoots()` 只知道后两种布局，于是
`importSeam("@deepseek-ai/dsh-llm")` 在加载期就抛 `ERR_MODULE_NOT_FOUND`。

现在还会从 `process.execPath` 和 `process.resourcesPath` 推导锚点
（`<install>/resources/app.asar/dsh/node_modules`、同级的 `.unpacked` 目录，
以及 macOS 的 `Contents/Resources` 写法）。Electron 的 `fs`/`import` 能直接读 asar 路径，
不需要解包。

### 2. `lib/compress.mjs` — 协商 `reasoningEffort`，而不是硬编码 `"off"`

dsh 会拿 `reasoningEffort` 去校验适配器声明的元数据，不支持就**直接拒绝**，不做裁剪。
压缩器原本硬编码 `"off"`，为的是不让压缩器把输出预算花在自己的思考上——但这在一个
没有 `off` 档的模型上会直接失败：

```
provider "workbuddy" model "deepseek-v4.1-flash" does not support reasoning effort "off"
```

`pickReasoningEffort()` 现在先问 `ctx.llm.resolveModelInfo(provider, model)`
这个具体模型接受哪些档位，按 `off` → `none` → `minimal` → 最低可用档 依次选择；
探测失败、或模型压根没有声明 reasoning 元数据时，**完全不传** `reasoningEffort`。

### 3. `lib/compress.mjs` — 补一条开头的 system 消息

有些网关要求 transcript 以 system 行开头。WorkBuddy 对首条消息是用户回合的请求返回
`400` + code `11128`（“first message is not system prompt”，伪装成安全策略拦截）。
压缩器是单用户回合调用，所以每次压缩都被拒：

```
workbuddy: 请求被拦截，请重新发送 first message is not system prompt
```

现在调用前会插入 `Llm.createSystemMessage(COMPRESSOR_SYSTEM)`。system 文本本身不带
任务指令（指令在用户消息里），它只是用来满足网关。压缩调用依然不是 agent-loop 请求，
所以 `llm/stream` waterfall 会忽略它，不会递归。

## 实测生效

在桌面版上，`provider: workbuddy`、`model: deepseek-v4.1-flash`、`profile: full`：

```
thinking-compressed  charsIn 1354 → charsOut 575   ms 4402  finish "stop"  retried false
block-replaced       charsIn 1354 → charsOut 575

thinking-compressed  charsIn 4449 → charsOut 566   ms 6052  finish "stop"  retried false
block-replaced       charsIn 4449 → charsOut 566
```

两次都是单次成功（没有 `retried`，没有 `fallback`），且会话日志里已提交的
`assistant/message` 事件包含的是压缩后的节点链，而不是原始草稿。
打完补丁后 `selftest.mjs` 30/30 通过。
