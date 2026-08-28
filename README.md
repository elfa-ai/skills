# Elfa AI Skills

Real-time crypto social intelligence and automated condition-engine skills for AI agents. Track trending tokens, surface narratives, search mentions, run market analysis, and build automated trigger-based workflows.

Works with **Claude Code**, **OpenCode**, **Cursor**, **GitHub Copilot**, **Codex**, and any tool that supports the [Agent Skills](https://agentskills.io) standard.

## Installation

### Quick (any agent)

```bash
npx skills add elfa-ai/skills
```

Installs the skill via the [Skills CLI](https://github.com/vercel-labs/skills); works with Claude Code, Cursor, Windsurf, Codex, and [40+ other agents](https://github.com/vercel-labs/skills#supported-agents). Add `--global` to install for all projects. Run `npx skills update` to refresh.

### Manual (spec-compliant `.agents/skills/`)

For any agent that supports the [Agent Skills](https://agentskills.io) standard:

```bash
git clone https://github.com/elfa-ai/skills elfa-skills
cd elfa-skills

# Project-level (current directory only)
mkdir -p .agents/skills && cp -r skills/. .agents/skills/

# OR user-level (all projects)
mkdir -p ~/.agents/skills && cp -r skills/. ~/.agents/skills/
```

### Manual (agent-specific paths)

If your agent doesn't yet scan `.agents/skills/`, copy into its native skills directory:

| Agent | Project-level | User-level |
|---|---|---|
| Spec-compliant | `.agents/skills/` | `~/.agents/skills/` |
| Claude Code | `.claude/skills/` | `~/.claude/skills/` |
| OpenCode | `.opencode/skills/` | `~/.config/opencode/skills/` |

```bash
# Example: install all skills into Claude Code globally
mkdir -p ~/.claude/skills && cp -r skills/. ~/.claude/skills/
```

<details>
<summary>Cursor (uses rule format)</summary>

Cursor doesn't read `SKILL.md` directly; wrap each skill in an always-on rule:

```bash
for SKILL in skills/*/; do
  NAME=$(basename "$SKILL")
  mkdir -p .cursor/rules
  {
    echo '---'
    echo "description: \"$NAME\""
    echo 'alwaysApply: true'
    echo '---'
    cat "$SKILL/SKILL.md"
  } > ".cursor/rules/$NAME.mdc"
done
```

</details>

<details>
<summary>GitHub Copilot</summary>

```bash
for SKILL in skills/*/SKILL.md; do
  cat "$SKILL" >> .github/copilot-instructions.md
  echo >> .github/copilot-instructions.md
done
```

</details>

<details>
<summary>Codex (AGENTS.md)</summary>

```bash
for SKILL in skills/*/SKILL.md; do
  cat "$SKILL" >> AGENTS.md
  echo >> AGENTS.md
done
```

</details>

<details>
<summary>Claude Desktop (attach file)</summary>

1. Start a conversation in Claude Desktop
2. Attach the relevant `skills/<name>/SKILL.md` as a file
3. Ask Claude to use the skill

For a bundled `.skill` package with API docs and scripts included (currently `elfa-ai` only), run `./skills/elfa-ai/scripts/build-skill.sh` and attach the generated `dist/elfa-ai.skill` instead.

</details>

## Skills

| Skill | Description |
|---|---|
| [elfa-ai](skills/elfa-ai) | Crypto social intelligence + Auto condition engine — trending tokens, mentions, narratives, AI market analysis, and automated trigger workflows |

## Spec validation

Every skill in this repo follows the [Agent Skills](https://agentskills.io/specification.md) directory format: `SKILL.md` at the skill root, optional `scripts/`, `references/`, and `assets/` resources, and spec-compatible frontmatter.

Validate a skill before publishing changes:

```bash
uvx --from skills-ref agentskills validate ./skills/elfa-ai
```

## Get an API key

Grab a free key (1,000 credits) at **https://go.elfa.ai/claude-skills**

Set it as an environment variable:

```bash
export ELFA_API_KEY=your_key_here
```

Free tier works with most endpoints. Trending narratives and AI chat require Grow+ or PAYG; streaming AI chat requires PAYG or Enterprise — see the link above for details.

Alternatively, use **x402 keyless payments** to pay per request with USDC on Base, Arbitrum, Polygon, Avalanche, or Solana (no signup required). See the [x402 docs](https://docs.elfa.ai/x402-payments) for setup.

## Example prompts

```
Show me the top trending tokens in the last 24 hours
```

```
What are the top mentions for $SOL this week?
```

```
Get smart stats for @elaborateelf on Twitter
```

```
Give me a curl example for the keyword mentions endpoint
```

```
Help me integrate the Elfa trending tokens endpoint in TypeScript
```

```
Alert me when BTC crosses above 100k
```

```
Set up a recurring 4h portfolio check on BTC, ETH, and SOL
```

```
Create an Auto query that triggers when ETH RSI drops below 30 on the 1h chart
```

```
Help me build a multi-condition trigger for BTC + ETH breakout confirmation
```

## API endpoints

| Endpoint | Description |
|---|---|
| `/v2/aggregations/trending-tokens` | Trending tokens by mention count |
| `/v2/account/smart-stats` | Smart follower & engagement stats (legacy — removed 28 Oct 2026) |
| `/v2/data/top-mentions` | Top mentions for a ticker symbol |
| `/v2/data/keyword-mentions` | Search mentions by keyword |
| `/v2/data/event-summary` | AI event summaries (5 credits) |
| `/v2/data/trending-narratives` | Trending narrative clusters (5 credits) |
| `/v2/data/token-news` | Token-related news — X posts from accounts tagged as news sources |
| `/v2/data/market-events` | Impact-scored market events (Enterprise only — access-gated) |
| `/v2/aggregations/trending-cas/twitter` | Trending contract addresses (Twitter) |
| `/v2/aggregations/trending-cas/telegram` | Trending contract addresses (Telegram) |
| `/v2/chat` | AI chat — market analysis, token intros, account reviews |
| `/v2/chat/stream` | AI chat as an incremental SSE stream (PAYG / Enterprise) |
| `/v2/key-status` | API key usage & limits (free) |
| `/v2/ping` | Health check — no auth required (free) |

### Auto endpoints (Condition Engine)

| Endpoint | Description |
|---|---|
| `/v2/auto/chat` | Builder Chat — AI-assisted query building |
| `/v2/auto/queries/validate` | Validate EQL query and preview cost |
| `/v2/auto/queries` | Create and list Auto queries |
| `/v2/auto/queries/:queryId` | Poll query status (GET) |
| `/v2/auto/queries/:queryId/cancel` | Cancel an `active` query (POST) |
| `/v2/auto/queries/:queryId` | Delete a terminal query (DELETE — `triggered` / `expired` / `cancelled` / `failed` only) |
| `/v2/auto/queries/stream` | Stream notifications for **all** your queries on one connection (SSE, API-key only) |
| `/v2/auto/queries/:queryId/stream` | Stream notifications for a single query via SSE |
| `/v2/auto/queries/:queryId/sessions` | List/get LLM analysis sessions |
| `/v2/auto/queries/drafts` | Upsert, list, validate, convert, delete query drafts |
| `/v2/auto/executions` | List and get trigger execution records |
| `/v2/auto/validate-symbol/:exchange/:symbol` | Check whether a symbol is supported on a venue — pre-flight for `price`/`ta` data sources |

API key mode authenticates every `/v2/auto/*` route with `x-elfa-api-key` alone. x402 mode uses `x-elfa-agent-secret` on query lifecycle routes instead. See [Auto docs](https://docs.elfa.ai/auto/overview).

Full details at [docs.elfa.ai](https://docs.elfa.ai).

---

Powered by [Elfa AI](https://go.elfa.ai/claude-visit) · [Documentation](https://docs.elfa.ai) · [Auto Docs](https://docs.elfa.ai/auto/overview)
