# Channel taxonomy — where a receipt/invoice actually lives

Generalized from real production experience automating supplier document retrieval. Try channels in this order (cheapest/most reliable first). The Researcher should identify which channel applies to a given supplier, execute it, and record the exact working path in `businesses.retrieval_pattern` regardless of outcome (a documented failure is as valuable as a documented success — it stops the next run from repeating it).

## Channel A — email attachment (most common)
The document is a direct attachment on an email from the supplier. Prefer whatever mail access gives you the raw attachment bytes directly (a mail API's attachment-fetch endpoint, if available) over a generic mail connector that only exposes metadata/ids without bytes — many MCP mail connectors list attachments but do not export their content, in which case fall back to a browser: open the specific message, use the message's own "download attachment" control (not a preview/thumbnail click, which only opens a viewer), and let the file land in the normal downloads location.

Watch for attachments served as `application/octet-stream` instead of `application/pdf` — the obvious "download" button sometimes only works for recognized MIME types; look for a secondary hover/download affordance on the attachment chip.

## Channel B — hosted link inside an email (e.g. payment processor receipts)
Many billing emails (Stripe and similar) contain a direct link to a hosted PDF that can be fetched without authentication. Try a direct fetch first — it's the cheapest possible path, no browser needed.

**These links expire** (observed on the order of a few weeks). An expired link returns an error page, not the PDF. The fallback is always channel A: the same email almost always also carries the PDF as a plain attachment. Treat "hosted link failed" as a normal, expected branch to channel A, not a dead end.

## Channel C — web portal (the hard case, several sub-patterns)

**C1 — portal navigable in an authenticated browser session.** If a browser automation tool is available and already has an authenticated session with the supplier, navigate to the billing/invoices section and download directly. Look specifically for a "download by period" or bulk-export feature — much more efficient than fetching one invoice at a time.

**C2 — one-shot account setting that permanently upgrades the channel.** Many portals have a "send invoices by email" toggle. When found, flag it as a suggested one-time setup action — enabling it permanently converts that supplier from channel C to channel A for every future invoice. This is the single highest-leverage move available: always check for it.

**The generic "PDF gated by session" pattern**, useful whenever a direct fetch fails with an auth/redirect error and the mail connector can't export bytes:
1. Be on a normal HTML page of the *same origin* as the PDF (not inside a browser's native PDF viewer — those are typically not scriptable).
2. Trigger a same-origin fetch of the document URL with credentials included, turn the response into a blob, and trigger a download via a synthetic link click. This inherits the page's session cookies where a bare `curl`/fetch from outside the browser cannot.
3. Only report a short status back (e.g. "downloaded"), never the signed URL or raw bytes — some harnesses redact tool output containing base64 blobs or signed query strings as a privacy safeguard, but the side effect (the file landing on disk) still happens even when the output is redacted. Design the action to not depend on reading that output back.

**Capricious UI filters (e.g. a fragile date-range picker)**: if the filter state is reflected in the URL as a query parameter, prefer editing the URL parameter directly over fighting the widget.

**A portal action that opens an entirely new, uncontrolled browser tab/window** (common with "manage my subscription" buttons that hand off to a payment processor's own portal): if the automation tool only controls a specific tab/tab-group, it can lose track of the new tab. Treat this as a distinct blocker type rather than silently failing — it usually needs either a browser tool that controls all tabs, or one human click.

**Print-only receipts**: some portals render a receipt as an HTML page whose only "download" affordance is the browser's native print-to-PDF dialog (not a real download link). A native print dialog is generally not scriptable by a browser-automation extension, and a plain on-screen screenshot capture is usually not an acceptable substitute (not treated as an uploadable file, and lossy). This is the clearest case where a fuller browser-automation capability (one that can render a page to PDF programmatically) is needed rather than an extension-based approach — flag it as `missing_capability` if only a lighter-weight browser tool is available, rather than attempting a workaround that produces a bad result.

**Login-gated portals with no existing session**: if there is genuinely no authenticated session available and no credential to establish one, this is a one-shot human setup blocker (`login_required`) — not a dead end. Once a human logs in once, the supplier permanently moves to channel C1/auto.

## Channel D — human supplier, no invoice exists yet
Freelancers/small vendors who don't proactively send invoices: the invoice must be requested. Draft the request (email or message) but never send it (see guardrails) — leave it ready for the operator to review and send. Once sent (by the operator) and a reply arrives, it becomes channel A on the next sweep.

## Channel E — local deposit / photo
Physical receipts (restaurants, taxis, small purchases): expect the operator to drop these into a conventional local folder. Match by amount + date proximity. Also check any other locations the operator names (a shared drive folder, the desktop, etc.).

## Channel F — request-by-form, delivery DEFERRED by email
Some suppliers have no downloadable document at all: filling out a request form is the only interface, and the actual invoice arrives by email later (sometimes hours later, sometimes only after a triggering event like a completed trip — don't assume the stated SLA is exact, poll rather than wait for a theoretical date). Fill the form out fully but never submit it autonomously (see guardrails) — leave it ready, precisely described, for the operator to submit. Once actually submitted, treat the payment as `awaiting_async`; the next routine sweep of channel A will pick up the reply.

## Channel for credits — payout/settlement reports (not a per-transaction document)
Incoming payment-processor payouts (e.g. marketplace/payment-platform settlements) are typically **not** backed by a single per-transaction invoice. The correct supporting document is a periodic payout/reconciliation report from the processor's dashboard, covering a date range that includes the payout. Don't search for a 1:1 document for these — search for the report covering the right period, and match by `period_covered_from/to` overlapping the transaction's settlement date rather than an exact amount (a payout report also nets fees, so the report total may differ from any single transaction).

## General web search
Use as a last resort when the supplier/portal itself is unknown (to find the right billing portal URL, support docs, etc.) — never to guess a document's URL directly. If a specific URL 404s, that's a signal the site has been restructured; navigate from the real entry point instead of guessing further paths.
