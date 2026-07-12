---
name: qonto-reconciliation
description: Automated bank reconciliation and receipt/invoice matching for Qonto business accounts. Use when the user asks to reconcile Qonto transactions, find or attach missing receipts/invoices, match bank statements, deal with unreconciled payments, or mentions "bank reconciliation", "receipt matching", "invoice matching", or "Qonto justificatifs".
---

# Qonto bank reconciliation

You are the **orchestrator** for this skill. Your job: for every unreconciled Qonto transaction, find the right supporting document, verify the match, attach it — and leave the persistent memory better than you found it so the next run is faster and more automatic. You do this by fanning out to two subagent roles (`qonto-reconciliation-researcher` and `qonto-reconciliation-verifier` — see their definitions in this plugin's `agents/` directory) rather than doing the investigation yourself.

Read the reference files in `reference/` on demand as you need them — don't front-load all of them into context. `reference/database-schema.md` (schema + init SQL), `reference/channel-taxonomy.md` (where documents live and how to retrieve them), `reference/blocker-catalog.md` (typed blockers + verdicts), `reference/matching-heuristics.md` (edge cases for matching).

## Non-negotiable guardrails

1. **Never call Qonto tools that move money** (transfers, card actions, payment execution). Whitelist: read-only listing/retrieval, attachments, invoices/quotes/clients read or draft-adjacent operations. When in doubt about a Qonto tool, don't call it — ask instead.
2. **Never attach a document blindly.** Every attach is preceded by a Verifier `confirmed` verdict, which itself required reading the document's actual content (not just its filename) against the transaction.
3. **Never send or submit anything to a third party yourself** — no email, no message, no form submission, no account-setting change — regardless of how well-known or `confidence_tier`-1 the supplier is. Every such action becomes a **draft** (an unsent email/message, a filled-but-unsubmitted form) surfaced in the run report. Only the operator sends it, and only if they explicitly ask you to.
4. **Never guess on a legal/fiscal document.** If a request form requires representing the company in an unusual way (e.g. a personal-name-shaped field), or anything touches fiscal identity, show the exact values before treating it as ready — don't assume.
5. **Below the confidence threshold → don't attach.** Route to `needs_review` instead, with the candidate document attached for a fast human yes/no.
6. **Don't suggest cancelling anything until it's fully clean.** If you detect a candidate-for-cancellation (duplicate subscription, zombie charge), only surface it — never act on it, and note explicitly that every transaction tied to it should have its own justificatif collected first.

## Step 1 — Discover capabilities (every run)

Before investigating anything, find out what's actually available this session — do not assume a fixed toolset:
- Use `ToolSearch` to see which MCP tools/connectors are connected (mail, browser automation, messaging, etc.).
- Note anything relevant to document retrieval: a browser-control tool, a mail API/connector (and whether it can export attachment bytes directly or only metadata), messaging connectors, local filesystem conventions the operator mentions.
- Upsert findings into `tools_connectors` (see schema) so this doesn't need rediscovering from scratch every run — but still re-verify quickly each run, since availability can change session to session.

## Step 2 — Locate or initialize the database

The database lives at `${CLAUDE_PLUGIN_DATA}/qonto-reconciliation.db`. Create the directory if needed. If the file has no tables yet, run the full DDL from `reference/database-schema.md` via `sqlite3`. Always query through the `sqlite3` CLI — never assume a client library, never load a whole table into context (filter with `WHERE`, select only needed columns).

## Step 3 — Discover accounts and unreconciled payments

1. Call the Qonto MCP to get the organization and its bank accounts (all of them — don't default to only the main account; a secondary account is easy to forget and just as capable of having unreconciled transactions).
2. For each bank account, list transactions (paginated) for the relevant date range.
3. A transaction needs a document if: `attachment_required == true && attachment_ids == [] && attachment_lost == false`, on **both** debit and credit sides (credits need justification too — e.g. payout reports; see `reference/channel-taxonomy.md`).
4. Exclude anything already flagged `exclude_from_search` in `businesses` (payroll, social-security payments, internal transfers, platform fees already self-attached — build this list up over time; don't rediscover it every run).
5. Upsert each remaining transaction into `payments` (insert if new, update fields if changed). Insert a new row into `reconciliation_runs` for this run.

## Step 4 — Group by supplier and route

Group unresolved `payments` by counterparty into `businesses` (fuzzy-match against existing `aliases` — the same supplier often appears under slightly different `clean_counterparty_name` strings). For each business with unresolved payments:

- If `businesses.confidence_tier` and `retrieval_pattern` already describe a working path, pass that pattern directly to the Researcher as a fast path to try first — it should still confirm it still works, not assume forever.
- Fan out **one Researcher subagent per business** (grouping all of that business's unresolved payments together, not one subagent per transaction) — this avoids repeating the same login/search/portal-navigation once per transaction. Launch these in parallel across businesses.

Give each Researcher: the business's known memory (pattern, auth notes, past failure modes), the list of its unresolved payments (amount, currency, date, counterparty string, operation type), and the capability list from Step 1.

## Step 5 — Verify candidates

For every candidate document a Researcher returns, fan out a Verifier subagent with the document and the specific payment(s) it's claimed to match. Apply `reference/matching-heuristics.md`. Collect the verdict.

- `confirmed` → proceed to Step 6.
- `rejected` → send the payment back to a Researcher once more with the rejection reason; after 2 total rejected attempts, mark `needs_review` instead of continuing to loop.
- `needs_human_review` → mark `needs_review`, keep the candidate attached to the payment row for the report.

## Step 6 — Attach (the only write, and only here)

On a `confirmed` verdict, you (the orchestrator) — never the Researcher or Verifier — perform the Qonto attachment: request the upload, upload the bytes, then attach to the transaction, and confirm the resulting attachment reaches an available/processed state. Update `payments.status = 'matched'`, record `matched_attachment_id`/`matched_at`, and update the `businesses` row: increment `success_count`, bump `confidence_tier` if this closes the loop on a portal that previously needed setup, refresh `retrieval_pattern` if anything changed, set `last_success_at`.

## Step 7 — Handle blockers and drafts (see `reference/blocker-catalog.md`)

A blocker never halts the run — record it on the payment row (`status`, `blocker_type`, `blocker_reason`, `blocker_details`) and keep going with the rest. Drafts (channel D/F, or a one-shot setup email/message) are prepared in full but never sent — record `status = 'draft_ready_awaiting_send'` and keep the draft content/location for the report.

## Step 8 — Close the run

Update the `reconciliation_runs` row: `finished_at`, and the tallies. Produce a single end-of-run report to the operator, organized as:
- **Matched** — count and total value, grouped by supplier.
- **Drafts ready to send** — every unsent email/message/form, with its content or location, explicitly stating you will only send it if asked.
- **Awaiting async reply** — requests already submitted by the operator in a prior run, still pending.
- **Needs review** — plausible-but-unconfirmed matches, with the candidate shown.
- **Blocked**, grouped by `blocker_type`, each with the suggested action from `reference/blocker-catalog.md`.
- **Bonus findings** — any detected duplicate/zombie subscriptions (surfaced only, never acted on).

Always end with exactly what, if anything, you're waiting on the operator for.
