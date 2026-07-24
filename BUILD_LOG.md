# Build Log: Assignment 3 — n8n Automation (AlbaCars Recall & Safety Watch)

## Goal & scope decision
- Built a **daily recall-watch automation** for a used-car dealer: scan the cars in stock
  against live NHTSA safety data, summarise with an LLM, and post a Slack briefing.
- Chose it because it makes the three assignments **one product**: it reuses AutoScope's NHTSA
  safety data (A1) to serve the AlbaScope dealer (A2). Not a throwaway "fetch one URL" demo — it
  automates a real liability a dealer actually has.

## Stack & tooling
- **n8n Cloud** (free trial instance) — hand-in as a live instance *and* exported JSON.
- **APIs:** NHTSA vPIC (VIN decode) + NHTSA Recalls (both live, keyless), Anthropic Messages API
  (the LLM briefing), Slack Incoming Webhook (delivery).
- Workflow authored as JSON and imported (precise + reviewable), not click-built.
- **AI tooling:** **ChatGPT** for ideation and scoping → workflow designed, authored (JSON), and
  verified in **Claude Code** (live NHTSA calls tested before shipping).

## Key decisions & trade-offs
- **Sample inventory in a Code node**, clearly labelled, instead of reading the live Supabase
  `cars` table — so a reviewer can import and run it in 30 seconds with no access to my private
  DB (RLS would block them anyway). The production swap is a one-node change, documented.
- **HTTP Request to Anthropic** rather than the LangChain Anthropic node — version-proof across
  n8n releases and keeps the API key in an n8n credential, never in the exported JSON.
- **Idempotency via n8n workflow static data** rather than a Google Sheet — persistent across
  runs, zero extra credentials, still earns the bonus.
- **Two NHTSA endpoints fused** (decode + recalls) to satisfy multi-source merging without adding
  another credential to set up.
- **Both branches converge on one Slack node** — cleaner than duplicating the delivery node.

## Hard parts / dead ends
- **Keeping the run reliable regardless of VIN validity:** the recall query is built from the
  dealer's own make/model/year, and VIN decode is treated as best-effort enrichment — so an
  unknown VIN enriches less but never breaks the recall lookup.
- **NHTSA quirk:** recalls come back under a lowercase `results` key while vPIC uses `Results` —
  parsed each explicitly.
- **Error handling on purpose:** every HTTP node uses `continueOnFail` + retry/backoff, and the
  Code nodes default missing fields, so one flaky API response degrades gracefully instead of
  killing the run.

## How I verified it works
- Confirmed all 7 sample vehicles return **real** recalls from the live API before shipping
  (Camry 2012 → 2, Accord 2013 → 5, Escape 2013 → 18, Altima 2013 → 12, Optima 2013 → 9,
  Sonata 2015 → 9, Grand Cherokee 2014 → 19), and that VIN decode resolves correctly.
- Ran the workflow end-to-end: report compiles, IF branches, LLM briefing generates, Slack
  message posts.
- **Idempotency:** second run reports 0 new recalls and posts the all-clear branch.

## Known limitations
- Inventory is labelled sample data; production reads Supabase.
- NHTSA data is US-market (same dataset as AutoScope).
- Free-tier n8n Cloud instance (1000 executions / 14 days).

## Time spent
- Design + workflow JSON: ~50m · API verification: ~15m · Docs: ~20m.
