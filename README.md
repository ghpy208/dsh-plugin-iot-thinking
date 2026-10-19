# dsh-plugin-iot-thinking

IoT thinking-block compression for **DeepSeek Harness 0.2.x**, packaged as an
**external bundle** (installable from Settings → 添加插件 with an absolute local
path, or with `dsh plugin add`).

> **Fork notice.** This is a patched fork of
> [`Roninton/IoT-items-of-thoughts`](https://github.com/Roninton/IoT-items-of-thoughts)
> (`dsh-plugin-iot-thinking/`). Three fixes were needed to run it on the DSH
> **desktop (Electron) build** with a **WorkBuddy-routed** model; upstream works
> for the CLI/npm layout. See [Patches in this fork](#patches-in-this-fork).

## The problem it fixes

The previous `iot-compaction` backend used `ctx.compaction`. dsh's compaction
replaces a **history range** and appends a checkpoint message. The model sees
that checkpoint as a new user turn:

> 上下文已压缩 · 已压缩 10 条历史记录（约 5002 tokens）

and then thinks:

> The conversation has been compacted. I need to figure out where we are…

That is not IoT's rule. IoT says: **when a thinking block ends, compress that
block and return the compact block in place.** The body, tool calls, usage and
finish reason stay native.

## How this plugin does it

dsh exposes `llm/stream` as a **waterfall** around every model call. Adapters
emit raw chunks; dsh assembles the committed assistant message from whatever the
waterfall yields. This plugin:

1. passes `block-start` through (the thinking block starts normally);
2. holds `reasoning-delta` chunks per block index;
3. at the reasoning block's `block-end`, calls the IoT compressor with the full
   draft;
4. yields **one compact `reasoning-delta`** and a **rewritten `block-end`**;
5. passes text deltas, tool-call deltas, usage and finish through untouched.

Because the replacement happens **before** the assistant message is committed,
there is no second message, no checkpoint, no `compaction/start`,
`compaction/end`, or `compaction` service. It also works inside preset sessions,
because the hook is at the LLM layer rather than a preset-local compaction realm.

If compression fails, is disabled, or the result is empty / not shorter than the
draft, the original deltas and block are yielded unchanged.

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

## Install

### From the UI

Settings → Plugins → **添加插件** → enter the absolute path:

```text
D:\_playground\workspace\IoT-items-of-thoughts\dsh-plugin-iot-thinking
```

The package declares `dsh.bundle.patch`, so the plugin manager adds it to the
profile's `dsh.profile.bundles` and its `cordis.patch.yml` row loads
automatically.

### From the CLI

```powershell
dsh plugin --profile web add "D:\_playground\workspace\IoT-items-of-thoughts\dsh-plugin-iot-thinking"
```

Restart `dsh web` once after installing (the host module is new). The client
half is served from `exports["./client"]`.

## Switch and limits

Settings → Plugins → **IoT 思考压缩**:

- **已启用 / 已关闭** — the live switch;
- **最小压缩字符数** — drafts shorter than this are left alone (default 400);
- **最大输入字符数** — drafts longer than this are left alone (default 24000);
- **节点输出上限** — compressor `maxTokens` (default 2000);
- **最多节点数** — hard cap on the node chain (default 8);
- **最多 [中性] 节点** — extra `[中性]` nodes are merged (default 2; `0` drops them);
- optional **provider / model** — leave blank to follow the current request's
  route, or set another provider/model registered in DSH.

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

The switch writes:

```text
$DSH_HOME/plugin-data/iot-thinking/config.json
```

and the stream hook reads it on every request (mtime cache), so toggling takes
effect on the **next model request** — no restart.

## Evidence

The only reliable proof that a draft was intercepted is the plugin ledger:

```text
$DSH_HOME/plugin-data/iot-thinking/events.jsonl
```

Events: `constructed`, `block-replaced`, `block-kept`, `block-skipped`,
`block-failed`, `thinking-compressed`, `thinking-failed`, `thinking-degenerate`.

Original drafts are archived before replacement:

```text
$DSH_HOME/plugin-data/iot-thinking/originals/<sessionId>/<time>-block<index>.txt
```

dsh stores whatever chunks the waterfall yields, so once a block is replaced the
original draft exists only in this archive.

## Files

| File | Role |
|---|---|
| `index.mjs` | host plugin: `llm/stream` hook + config API |
| `lib/stream.mjs` | pure chunk transformer (hold → compress → single delta + block-end) |
| `lib/compress.mjs` | auxiliary `ctx.llm.stream()` call(s), low-temperature retry, quality gate, deterministic fallback |
| `lib/think.mjs` | IoT instruction, prompt wrapper, draft sanitization, degeneration detection, output cleanup |
| `lib/config-store.mjs` | live JSON config (row defaults + user file) |
| `lib/config-api.mjs` | exact HTTP routes for the Settings card |
| `lib/ledger.mjs` | events ledger + original-draft archive |
| `lib/client.js` | Settings → Plugins → IoT 思考压缩 card |
| `selftest.mjs` | pure transformer tests (no dsh boot, no model) |

## Self-test

```powershell
cd D:\_playground\workspace\IoT-items-of-thoughts\dsh-plugin-iot-thinking
node selftest.mjs
```

Covers: replacement of a completed block, disabled pass-through, non-agent-loop
pass-through, `minChars`, compressor failure, empty/longer output fallback,
multiple blocks, aborted streams, block-end without deltas, output cleanup,
degenerate input/output detection, strict retry, and deterministic fallback.

## Known limits

1. **Latency at block end.** The reasoning block is shown as "thinking" while
   the draft is buffered, then the compact text appears. Streaming the compact
   text while compressing is deliberately out of scope ("压缩后返回").
2. **Extra model call per compressed block.** The compressor uses the current
   route by default; set a cheaper provider/model in the card if needed.
3. **Not a security boundary.** The plugin runs with the host's permissions and
   can read model output; install only from a source you trust.
4. **The original draft is not in the session log.** It is archived under
   `plugin-data/iot-thinking/originals/`; keep that directory if you need audit.
5. **Degenerate retry costs one more compressor call.** Only when the quality
   gate rejects the first output; the deterministic fallback itself is local.
6. **The fallback is lossy.** If both attempts are degenerate, it keeps at most
   one fact line plus a plan line; the full draft remains in `originals/`.

## Patches in this fork

All three were found by running the plugin on the **DSH desktop (Electron)
build** against a **WorkBuddy-routed** model. Each one made the plugin fail
silently-but-safely: the original thinking block was passed through untouched,
so the plugin looked installed while compressing nothing.

### 1. `lib/seams.mjs` — resolve host packages from the desktop install

The desktop build ships the host `@deepseek-ai/*` packages **inside
`resources/app.asar`**, not in `$DSH_HOME/profiles/node_modules` or a global npm
root. `defaultNodeModulesRoots()` only knew those two layouts, so
`importSeam("@deepseek-ai/dsh-llm")` threw `ERR_MODULE_NOT_FOUND` at load time.

Now it also derives anchors from `process.execPath` and
`process.resourcesPath` (`<install>/resources/app.asar/dsh/node_modules`, the
`.unpacked` sibling, and the macOS `Contents/Resources` spelling). Electron's
`fs`/`import` can read asar paths directly, so no extraction is needed.

### 2. `lib/compress.mjs` — negotiate `reasoningEffort` instead of hardcoding `"off"`

dsh validates `reasoningEffort` against the adapter's advertised metadata and
**rejects** rather than clamping. The compressor hardcoded `"off"` to keep the
compressor from spending its output budget on thinking — which fails outright on
a model that advertises no `off` rung:

```
provider "workbuddy" model "deepseek-v4.1-flash" does not support reasoning effort "off"
```

`pickReasoningEffort()` now asks `ctx.llm.resolveModelInfo(provider, model)` what
this exact model accepts and picks `off` → `none` → `minimal` → cheapest
advertised rung, sending **no** `reasoningEffort` at all when the probe fails or
the model advertises no reasoning metadata.

### 3. `lib/compress.mjs` — send a leading system message

Some gateways require the transcript to open with a system row. WorkBuddy
returns `400` + code `11128` ("first message is not system prompt", dressed up as
a safety-policy block) for a request whose first message is a user turn. The
compressor is a single-user-turn call, so every compression was rejected:

```
workbuddy: 请求被拦截，请重新发送 first message is not system prompt
```

The call now prepends `Llm.createSystemMessage(COMPRESSOR_SYSTEM)`. The system
text carries no task instruction — the user message does — it exists only to
satisfy the gateway. The compressor is still not an agent-loop request, so the
`llm/stream` waterfall ignores it and there is no recursion.

## Verified working

On the desktop build with `provider: workbuddy`, `model: deepseek-v4.1-flash`,
`profile: full`:

```
thinking-compressed  charsIn 1354 → charsOut 575   ms 4402  finish "stop"  retried false
block-replaced       charsIn 1354 → charsOut 575

thinking-compressed  charsIn 4449 → charsOut 566   ms 6052  finish "stop"  retried false
block-replaced       charsIn 4449 → charsOut 566
```

Both are single-pass (no `retried`, no `fallback`), and the committed
`assistant/message` events in the session log contain the compacted node chain
rather than the original draft. `selftest.mjs` passes 30/30 after the patches.
