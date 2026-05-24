# 配置指南（中文）

English version: [configuration.md](configuration.md)

## 快速开始

使用 OpenClaw 的插件安装器安装插件：

```bash
openclaw plugins install @martian-engineering/lossless-claw
```

如果你是在本地 OpenClaw checkout 中运行：

```bash
pnpm openclaw plugins install @martian-engineering/lossless-claw
```

如果你是在本地开发这个插件，并希望用 `--link` 方式直接挂载工作目录：

```bash
cd /path/to/lossless-claw-enhanced
npm install
openclaw plugins install --link .
```

`openclaw plugins install` 会自动处理插件注册、启用和 slot 选择。
但 `--link` 模式不会替你安装这个插件自己的 npm 依赖，所以对 fresh clone 必须先在插件目录里执行一次 `npm install`。

建议设置这些环境变量：

```bash
export LCM_FRESH_TAIL_COUNT=32
export LCM_INCREMENTAL_MAX_DEPTH=-1
```

然后重启 OpenClaw。

### OpenClaw 2026.5.x 配置校验

部分 OpenClaw 2026.5.x 版本会在加载插件 schema 之前先校验
`openclaw.json`。在这些版本里，把自定义插件配置写到
`plugins.entries.lossless-claw.config` 下面，可能会报：

```text
OpenClaw config is invalid: ~/.openclaw/openclaw.json
  x plugins: Invalid input
```

遇到这个错误时，建议 `openclaw.json` 里只保留插件启用和
context-engine slot，LCM 参数改用环境变量：

```bash
export LCM_SUMMARY_MODEL=minimax/MiniMax-M2.5
export LCM_EXPANSION_MODEL=minimax/MiniMax-M2.5
# 如果模型名没有 provider 前缀，再加：
export LCM_SUMMARY_PROVIDER=minimax
export LCM_EXPANSION_PROVIDER=minimax
```

当 OpenClaw host 接受 plugin config object 时，插件仍然支持
`summaryModel`、`summaryProvider`、`expansionModel`、`expansionProvider`。
对严格的 2026.5.x 配置校验器来说，环境变量是兼容路径。

## 调优指南

### 上下文阈值

`LCM_CONTEXT_THRESHOLD`（默认 `0.75`）控制 compaction 何时触发，值是模型上下文窗口的一个比例。

- **更低的值**（例如 `0.5`）会更早触发 compaction，保持上下文更小，但会增加用于摘要的 LLM 调用次数。
- **更高的值**（例如 `0.85`）允许会话先增长更久再压缩，能减少摘要成本，但在模型输出较长时更容易接近上下文上限。

对大多数场景来说，`0.75` 是一个比较均衡的默认值。

### Fresh tail count

`LCM_FRESH_TAIL_COUNT`（默认 `32`）表示最近多少条消息永远不会被压缩。这些原始消息能给模型提供最近对话的连续性。

- **较小的值**（例如 `8` 到 `16`）能省上下文空间，把更多预算留给摘要。
- **较大的值**（例如 `32` 到 `64`）能保留更多最近语境，但会提高上下文的基础占用。

对带工具调用的 coding 对话，推荐 `32`。

### Leaf fanout

`LCM_LEAF_MIN_FANOUT`（默认 `8`）表示在 fresh tail 之外，至少要积累多少条原始消息，leaf pass 才会执行。

- 值越低，摘要生成越频繁，得到更多、更小的摘要。
- 值越高，摘要生成越少，但每次会覆盖更大的输入范围。

### Condensed fanout

`LCM_CONDENSED_MIN_FANOUT`（默认 `4`）控制同一深度的摘要要累积到多少个，才会继续向上凝聚成更高层摘要。

- 值越低，DAG 层级会更深，抽象层更多。
- 值越高，DAG 会更浅，但每层节点会更多。

### Incremental max depth

`LCM_INCREMENTAL_MAX_DEPTH`（默认 `0`）控制 leaf pass 之后是否自动继续做 condensation。

- **`0`**：只做 leaf summaries。更高层摘要只会在手动 `/compact` 或溢出时生成。
- **`1`**：每次 leaf pass 后，尝试把 d0 摘要合成为 d1。
- **`2+`**：继续向更深层自动凝聚，直到指定深度。
- **`-1`**：不设上限。每次 leaf pass 后按需一路向上凝聚。适合长时间运行的会话。

### Summary target tokens

`LCM_LEAF_TARGET_TOKENS`（默认 `1200`）和 `LCM_CONDENSED_TARGET_TOKENS`（默认 `2000`）控制摘要输出的目标长度。

- 值越大，保留的细节越多，但也会占用更多上下文。
- 值越小，摘要越激进，细节流失会更快。

实际输出长度仍取决于 LLM 本身，这两个值更像是 prompt 中给模型的目标提示。

### Leaf chunk tokens

`LCM_LEAF_CHUNK_TOKENS`（默认 `20000`）限制单次 leaf compaction 最多读取多少源材料。

- 值越大，每次 leaf pass 会吸收更多内容，生成更综合的摘要。
- 值越小，摘要会更频繁地产生，但每次覆盖的内容更少。
- 这个值也会影响 condensed pass 的最小输入阈值（它的 10%）。

## 模型选择

默认情况下，LCM 会使用父 OpenClaw 会话当前的模型来做摘要。你也可以覆盖它：

```bash
# 为 summarization 指定单独模型
export LCM_SUMMARY_MODEL=anthropic/claude-sonnet-4-20250514
export LCM_SUMMARY_PROVIDER=anthropic
export LCM_SUMMARY_BASE_URL=https://api.anthropic.com
```

如果你用更便宜或更快的模型做摘要，成本可能会下降，但摘要质量同样重要，因为低质量摘要会在更高层 condensation 中不断累积放大。

当系统同时存在多个来源时，compaction summarization 会按下面顺序解析模型：

1. `LCM_SUMMARY_MODEL` / `LCM_SUMMARY_PROVIDER`
2. 插件配置里的 `summaryModel` / `summaryProvider`
3. OpenClaw 默认的 compaction model/provider
4. 旧版每次调用传入的 model/provider 提示

如果 `summaryModel` 本身已经带了 provider 前缀，例如 `anthropic/claude-sonnet-4-20250514`，那么 `summaryProvider` 会被忽略。

## 会话控制

### 完全排除某些会话

使用 `ignoreSessionPatterns` 或 `LCM_IGNORE_SESSION_PATTERNS` 可以把低价值会话完全排除在 LCM 之外。命中的会话不会创建 conversation，不会写消息，也不会参与 compaction 或 delegated expansion grant。

- 匹配对象是完整的 session key。
- `*` 匹配除 `:` 以外的任意字符。
- `**` 匹配任意内容，包括 `:`。

示例：

```bash
export LCM_IGNORE_SESSION_PATTERNS=agent:*:cron:**,agent:main:subagent:**
```

### Stateless sessions

使用 `statelessSessionPatterns` 或 `LCM_STATELESS_SESSION_PATTERNS` 可以让某些会话只读不写。这对 sub-agent session 特别有用，因为它们通常使用像 `agent:<agentId>:subagent:<uuid>` 这样的真实 OpenClaw session key。

启用方式：

```bash
export LCM_SKIP_STATELESS_SESSIONS=true
```

当 session key 命中 stateless pattern 且开启 enforcement 后，LCM 会：

- 跳过 bootstrap imports
- 跳过 ingest 和 after-turn persistence
- 跳过 compaction writes
- 跳过 delegated expansion grant writes
- 仍然允许从已有的持久化上下文中做只读 assembly

示例：

```bash
export LCM_STATELESS_SESSION_PATTERNS=agent:*:subagent:**,agent:ops:subagent:**
export LCM_SKIP_STATELESS_SESSIONS=true
```

插件配置示例：

```json
{
  "plugins": {
    "entries": {
      "lossless-claw": {
        "config": {
          "ignoreSessionPatterns": [
            "agent:*:cron:**"
          ],
          "statelessSessionPatterns": [
            "agent:*:subagent:**",
            "agent:ops:subagent:**"
          ],
          "skipStatelessSessions": true
        }
      }
    }
  }
}
```

## TUI 会话窗口大小

`LCM_TUI_CONVERSATION_WINDOW_SIZE`（默认 `200`）控制 `lcm-tui` 在某个 session 已经绑定 `conversation_id` 时，每个 keyset-paged conversation window 会加载多少条消息。

- 值更小，面对超大对话时查询和渲染成本更低。
- 值更大，每页能看到更多上下文，但渲染时间也会更长。

## 数据库管理

SQLite 数据库路径由 `LCM_DATABASE_PATH` 控制，默认是 `~/.openclaw/lcm.db`。

### 检查数据库

```bash
sqlite3 ~/.openclaw/lcm.db

# 统计 conversation 数量
SELECT COUNT(*) FROM conversations;

# 查看某个 conversation 的 context items
SELECT * FROM context_items WHERE conversation_id = 1 ORDER BY ordinal;

# 查看不同 summary depth 的分布
SELECT depth, COUNT(*) FROM summaries GROUP BY depth;

# 找出较大的 summaries
SELECT summary_id, depth, token_count FROM summaries ORDER BY token_count DESC LIMIT 10;
```

### 备份

数据库就是一个单文件，可以直接这样备份：

```bash
cp ~/.openclaw/lcm.db ~/.openclaw/lcm.db.backup
```

也可以用 SQLite 的在线备份：

```bash
sqlite3 ~/.openclaw/lcm.db ".backup ~/.openclaw/lcm.db.backup"
```
