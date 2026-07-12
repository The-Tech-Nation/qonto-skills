# Qonto Bank Reconciliation (Claude Code plugin)

Automated bank reconciliation and receipt/invoice matching for Qonto business accounts. For every unreconciled transaction (debit or credit), it researches every available channel — email attachments, hosted links, web portals, local files, web search — finds the right supporting document, verifies the match, and attaches it via the Qonto MCP. Learns supplier-specific retrieval patterns over time in a local, persistent database so each run gets faster and more automatic than the last.

## What's in this plugin

- `skills/qonto-reconciliation/` — the orchestrator skill (`SKILL.md`) plus reference material (`reference/`) covering the retrieval-channel taxonomy, a typed blocker catalog, matching heuristics/edge cases, and the database schema.
- `agents/researcher.md` — subagent role that investigates a single supplier across all available channels until it finds a document or hits a genuine, typed blocker. No Qonto write access.
- `agents/verifier.md` — subagent role that verifies a candidate document against a transaction. Read-only, no network, no Qonto access at all.
- `.mcp.json` — bundles the official Qonto MCP server (`https://mcp.qonto.com/mcp`, OAuth). No other MCP servers are bundled — connect whatever else you use (mail, messaging, browser automation) yourself; the skill discovers and adapts to what's actually connected in your session rather than assuming a fixed toolset.

## Guardrails (see `SKILL.md` for the full list)

- Never calls Qonto tools that move money (transfers, cards, payments) — read/attachment/invoice tools only.
- Never attaches a document without a verified match.
- **Never sends or submits anything on your behalf** — no emails, no messages, no form submissions, ever, regardless of how well-known a supplier is. Everything that would need to go out is prepared as a draft and surfaced to you to send yourself.
- Never guesses on legal/fiscal documents — shows exact values before treating anything as ready if company identity is involved.
- Detects (but never acts on) potential duplicate/zombie subscriptions as a bonus finding.

## Persistent memory

State lives in `${CLAUDE_PLUGIN_DATA}/qonto-reconciliation.db` (SQLite), local to your machine, and survives plugin updates. It tracks reconciliation runs, per-transaction status, learned per-supplier retrieval patterns, and discovered tool/connector capabilities — all initialized automatically on first run.

## Requirements

Nothing beyond what Claude Code already provides: the `sqlite3` CLI (for the memory database) and a `pdftotext`-equivalent (for document verification) are expected to be available on the host machine. No bundled scripts, no npm/pip packages.
