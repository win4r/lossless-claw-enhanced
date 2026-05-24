# 🦞lossless-claw-enhanced

> Enhanced fork of [Martian-Engineering/lossless-claw](https://github.com/Martian-Engineering/lossless-claw) — fixes CJK token estimation and cherry-picks critical upstream bug fixes for production reliability.

Lossless Context Management plugin for [OpenClaw](https://github.com/openclaw/openclaw), based on the [LCM paper](https://papers.voltropy.com/LCM) from [Voltropy](https://x.com/Voltropy). Replaces OpenClaw's built-in sliding-window compaction with a DAG-based summarization system that preserves every message while keeping active context within model token limits.

中文配置文档: [docs/configuration.zh-CN.md](docs/configuration.zh-CN.md)

## Video Tutorial

> Full walkthrough: installation, configuration, and hybrid retrieval internals.

[![YouTube Video](https://img.shields.io/badge/YouTube-Watch%20Now-red?style=for-the-badge&logo=youtube)](https://youtu.be/m21PNaIW3N4)
**https://youtu.be/m21PNaIW3N4**

[![Bilibili Video](https://img.shields.io/badge/Bilibili-Watch%20Now-00A1D6?style=for-the-badge&logo=bilibili&logoColor=white)](https://www.bilibili.com/video/BV1MKXQBRE9d/)
**https://www.bilibili.com/video/BV1MKXQBRE9d/**



## What's enhanced

### CJK-Aware Token Estimation

The upstream plugin estimates tokens using `Math.ceil(text.length / 4)`, which assumes ~4 ASCII characters per token. This severely **underestimates by 2-4x** for CJK text (Chinese/Japanese/Korean), because each CJK character maps to ~1.5 tokens in modern tokenizers (cl100k_base, o200k_base).

**Impact of the upstream bug:**
- Compaction triggers too late, causing context window overflow
- Context assembly budgets are miscalculated
- Summary target sizes are wrong
- Large file interception misses CJK-heavy files

**What we fixed:**

| Character Type | Upstream (tokens/char) | Enhanced (tokens/char) | Correction |
|---|---|---|---|
| ASCII/Latin | 0.25 | 0.25 | unchanged |
| CJK (Chinese, Japanese, Korean) | 0.25 | 1.5 | **6x** |
| Emoji / Supplementary Plane | 0.5 | 2.0 | **4x** |

```
"这个项目的架构设计非常优秀" (14 CJK chars)
  Upstream:  ceil(14 / 4)   =  4 tokens  (wrong)
  Enhanced:  ceil(14 * 1.5) = 21 tokens  (accurate)
  Real (cl100k_base):         19 tokens
```

**Changes:**
- Shared `src/estimate-tokens.ts` with CJK/emoji-aware estimation
- Consolidated 5 duplicate `estimateTokens()` into a single import
- Idempotent migration recalculates `token_count` for existing CJK messages and summaries on upgrade (pure ASCII rows are not touched)
- 18 test cases (10 estimation + 8 migration)

### Cherry-picked Upstream Bug Fixes

| PR | Fix | Why it matters |
|---|---|---|
| [#178](https://github.com/Martian-Engineering/lossless-claw/pull/178) | Prevent false-positive auth errors in `stripAuthErrors()` | Conversations discussing "401 errors" or "API keys" caused the summarizer to falsely report auth failure, aborting compaction |
| [#190](https://github.com/Martian-Engineering/lossless-claw/pull/190) | Detect session file rotation in bootstrap | After `/reset` or session rotation, compaction never triggered on the new session — context grew unbounded |
| [#172](https://github.com/Martian-Engineering/lossless-claw/pull/172) | Skip ingesting empty error/aborted assistant messages | API 500s produced empty messages that accumulated, creating a feedback loop that permanently broke the agent |

All cherry-picks were reviewed by OpenAI Codex with 3 additional fixes applied:
- `parent_id` → `parent_summary_id` column name correction in session rotation purge
- FTS table operations guarded with try/catch for no-FTS runtimes
- CJK migration reordered before `backfillSummaryMetadata` so derived fields use corrected values

## Install

```bash
# Clone and install (link mode — picks up code changes instantly)
git clone https://github.com/win4r/lossless-claw-enhanced.git
cd lossless-claw-enhanced
npm install
openclaw plugins install --link .

# Or copy install (snapshot, won't pick up later changes)
openclaw plugins install ./lossless-claw-enhanced
```

`--link` mode does not install this plugin's npm dependencies for you, so run
`npm install` in the plugin directory first on a fresh clone.

### Configure OpenClaw

After installation, set the context engine slot:

```json
{
  "plugins": {
    "slots": {
      "contextEngine": "lossless-claw"
    },
    "entries": {
      "lossless-claw": {
        "enabled": true,
        "config": {
          "freshTailCount": 32,
          "contextThreshold": 0.75,
          "incrementalMaxDepth": -1,
          "ignoreSessionPatterns": [
            "agent:*:cron:**",
            "agent:*:subagent:**"
          ],
          "summaryModel": "anthropic/claude-haiku-4-5"
        }
      }
    }
  }
}
```

Restart the gateway after configuration changes:

```bash
openclaw gateway restart
```

#### OpenClaw 2026.5.x config validation

Some OpenClaw 2026.5.x builds validate `openclaw.json` before plugin schemas are
loaded. In that case, putting custom plugin config under
`plugins.entries.lossless-claw.config` can fail with:

```text
OpenClaw config is invalid: ~/.openclaw/openclaw.json
  x plugins: Invalid input
```

If you see that error, keep only the slot/enablement settings in
`openclaw.json` and set LCM tuning through environment variables instead. For
example:

```bash
export LCM_SUMMARY_MODEL=minimax/MiniMax-M2.5
export LCM_EXPANSION_MODEL=minimax/MiniMax-M2.5
# Optional when the model string does not include a provider prefix:
export LCM_SUMMARY_PROVIDER=minimax
export LCM_EXPANSION_PROVIDER=minimax
```

The plugin still supports `summaryModel`, `summaryProvider`, `expansionModel`,
and `expansionProvider` when the OpenClaw host accepts plugin config objects;
environment variables are the compatible path for strict 2026.5.x config
validators.

### Update to latest

```bash
cd /path/to/lossless-claw-enhanced
git pull origin main

# If using --link install, just restart the gateway:
openclaw gateway restart

# If using copy install, re-install:
openclaw plugins install /path/to/lossless-claw-enhanced
openclaw gateway restart
```

## Upstream compatibility

Tracks [Martian-Engineering/lossless-claw](https://github.com/Martian-Engineering/lossless-claw) `main` branch. To sync upstream changes:

```bash
cd /path/to/lossless-claw-enhanced
git fetch upstream
git merge upstream/main
```

## What it does

When a conversation grows beyond the model's context window, OpenClaw normally truncates older messages. LCM instead:

1. **Persists every message** in a SQLite database, organized by conversation
2. **Summarizes chunks** of older messages into summaries using your configured LLM
3. **Condenses summaries** into higher-level nodes as they accumulate, forming a DAG (directed acyclic graph)
4. **Assembles context** each turn by combining summaries + recent raw messages
5. **Provides tools** (`lcm_grep`, `lcm_describe`, `lcm_expand`) so agents can search and recall details from compacted history

Nothing is lost. Raw messages stay in the database. Summaries link back to their source messages. Agents can drill into any summary to recover the original detail.

## Configuration

LCM is configured through plugin config and environment variables. Environment variables take precedence.

### Key parameters

| Variable | Default | Description |
|----------|---------|-------------|
| `LCM_CONTEXT_THRESHOLD` | `0.75` | Fraction of context window that triggers compaction |
| `LCM_FRESH_TAIL_COUNT` | `32` | Messages protected from compaction |
| `LCM_INCREMENTAL_MAX_DEPTH` | `0` | Compaction cascade depth (`-1` = unlimited) |
| `LCM_LEAF_CHUNK_TOKENS` | `20000` | Max source tokens per leaf compaction |
| `LCM_SUMMARY_MODEL` | `""` | Model override for summarization, use `provider/model` format (see below) |
| `LCM_SUMMARY_PROVIDER` | `""` | Legacy provider override; prefer `provider/model` in `LCM_SUMMARY_MODEL` instead |
| `LCM_IGNORE_SESSION_PATTERNS` | `""` | Glob patterns to exclude from LCM |
| `LCM_DATABASE_PATH` | `~/.openclaw/lcm.db` | SQLite database path |

See upstream [README](https://github.com/Martian-Engineering/lossless-claw#configuration) for the full configuration reference.

### Summarization model

LCM uses an LLM to generate summaries during compaction. You can use **any model from any provider configured in OpenClaw** — there is no hardcoded provider list.

**Configuration priority** (highest to lowest):

1. Environment variables `LCM_SUMMARY_MODEL` (+ optional `LCM_SUMMARY_PROVIDER`)
2. Plugin config `summaryModel` (+ optional `summaryProvider`)
3. `agents.defaults.compaction.model` in OpenClaw gateway config
4. Falls back to the current agent's session model/provider

**Format:** Use the standard OpenClaw `provider/model` string:

```jsonc
"summaryModel": "anthropic/claude-haiku-4-5"
```

**Examples with different providers:**

```jsonc
// Anthropic (recommended — fast, cheap, good at summarization)
"summaryModel": "anthropic/claude-haiku-4-5"

// OpenAI
"summaryModel": "openai/gpt-4o-mini"

// Google
"summaryModel": "google/gemini-2.5-flash"

// DeepSeek
"summaryModel": "deepseek/deepseek-chat"

// Or via environment variable
// LCM_SUMMARY_MODEL=anthropic/claude-haiku-4-5
```

> **Note:** Model IDs depend on your OpenClaw provider configuration. Check your gateway's model catalog (`openclaw agents list` or `agents.defaults.models` in config) for available model IDs. The examples above use common upstream provider IDs which may differ from aliased names in your setup.

> **Note:** `LCM_SUMMARY_PROVIDER` is a legacy env var for when the model string does not include a provider prefix. Prefer the `provider/model` format instead.

**Choosing a model:** Summarization is a high-volume, low-complexity task — a fast, cheap model works best. We recommend Anthropic Haiku or a similar small/fast model. Using a large model (Opus, GPT-4o) works but adds cost and latency with no meaningful quality gain for compaction.

## Development

```bash
# Install dependencies
npm install

# Run tests
npx vitest run --dir test

# Run a specific test
npx vitest run test/estimate-tokens.test.ts
```

### Project structure (enhanced files)

```
src/
  estimate-tokens.ts         # [NEW] CJK-aware token estimation (shared module)
  engine.ts                  # [MODIFIED] Import shared estimator + session rotation fix + empty message skip
  assembler.ts               # [MODIFIED] Import shared estimator + empty assistant skip
  compaction.ts              # [MODIFIED] Import shared estimator
  retrieval.ts               # [MODIFIED] Import shared estimator
  summarize.ts               # [MODIFIED] Import shared estimator + auth false-positive fix + recall-accuracy prompts
  db/
    migration.ts             # [MODIFIED] CJK token recount migration
test/
  estimate-tokens.test.ts    # [NEW] 10 CJK estimation tests
  cjk-token-recount.test.ts  # [NEW] 8 migration tests
```

## License

MIT
