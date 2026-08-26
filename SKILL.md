---
name: deal-inspector
description: Pull deal/pipeline data from Sigma and cross-reference it with Gong call history to produce deal-health reports. Two workflows -- (1) Territory Pulse Report: a one-Word-doc-per-deal packet covering every big deal (ACV/stage threshold) across Marisa's SE roster, with a snapshot, plain-English recap, gap list, and a Green/Yellow/Red gut check per deal -- this is the flagship recurring routine, run whenever she asks for a pipeline review, territory scan, "big deals" check, or wants this run regularly (weekly, pre-forecast-call, etc). (2) Single-deal deep dive: a full MEDPICC report for one named opportunity/account, for when she asks to "inspect", "check on", or get a "health check" on a specific deal. Trigger on words like deal, opp, pipeline, territory, big deals, risk, health, MEDPICC, gut check, or objections, even without the phrase "deal inspector".
---

# Deal Inspector

Produces deal-health reports by combining Salesforce pipeline data with Gong call history, both queried live from Sigma's `Sales` data model. There are two workflows:

- **Territory Pulse Report** (primary/recurring) — every qualifying deal across Marisa's SE roster, one page per deal, in a single Word doc. This is the routine she runs regularly.
- **Single-deal deep dive** — one full MEDPICC report for a specific named opportunity, on request.

Default to the Territory Pulse Report when the ask is about "big deals," "the territory," "pipeline," or is unspecific ("run my deal review"). Default to the single-deal deep dive when a specific opportunity/account is named and the ask is about that deal specifically.

## Territory Pulse Report (primary workflow)

One Word doc, one page per deal, covering every deal that meets Marisa's scope criteria:

- **ACV ≥ $50,000**
- **Stage 3 or 4 only** (`3 - Establish Success Criteria` or `4 - Solution Evaluation` — excludes Stage 5 Solution Selection, Stage 6 Negotiation, Closed Won/Lost, 0-Nurture, and 1-Suspect). Marisa dropped 5 and 6 from scope since those are further along and lower-risk by nature of being closer to close.
- **SE is on her roster** — see `references/se_roster.md` for the current list of SE names. Confirm the roster is current before a full run if it's been a while; the reference file has the refresh query.

These thresholds are hers, not universal defaults — if she ever asks to adjust them (different ACV floor, different stage cutoff, a specific rep subset), treat that as a one-off filter change, not a reason to rewrite this section.

**At last count this scope was 100+ deals.** Do not query per-deal — it doesn't scale and burns huge numbers of tool calls. Instead:

1. **One query** against `Opportunities Enriched` for the full qualifying set — filter by SE roster (IN list), ACV, and stage, pull every field you'll need for the one-pagers (identity/ownership, deal mechanics, MEDPIC payload, competitor fields, next step, momentum fields, and `Opportunity Why Anything`/`Opportunity Why Buy Now`/`Opportunity Why Sigma` for the Use Case line — see the single-deal-deep-dive field list below, same fields apply here).
2. **One or two queries** against `Gong Calls Enriched` for the same set of Opportunity Guids (`WHERE "Opportunity Guid" IN (...)`), aggregated with `GROUP BY` to get call count, avg external sentiment, first call date, and last call date per deal — don't pull full transcripts or per-call rows at this scale, the aggregate is enough for a one-pager. A deal with zero rows back from this query is itself a finding (no Gong data linked to that opportunity — flag it as a data-hygiene gap, not a sentiment problem).
3. Synthesize each deal's one-pager from those two result sets plus your own reading of the free-text `Next Step` field (it's often a running log — read the most recent entries closely, don't just take the single latest line at face value, since deals sometimes have a placeholder next-step that's already stale).

### Forecast view (Rep / RVP / AVP)

Pull each deal's forecast categorization from the **Pipeline Forecasting** workbook (`workbookId: b07c84e2-04a8-402b-ac0f-424af4203352`), element **"AVP Scenario Planning"** (`elementId: sYLezBGNVv`), filtered on `"Opportunity Guid" IN (...)` for the same qualifying set (confirm the exact column ID via `describe` first — it was `MFSQBXNOZL` as of 8/26/2026, labeled "Opportunity Guid").

This is the one element on this workbook that actually carries all three tiers with real, populated data — confirmed 8/26/2026 by checking several deals. Don't use the plain "Frontline Forecast" or "RVP Scenario Planning" elements on this same workbook instead; their differently-named "RVP"/unprefixed Commit-Gut-Best columns came back null across every deal checked, while the equivalent AVP/RVP/Rep columns on `sYLezBGNVv` were populated for the same deals. If a future check finds `sYLezBGNVv` itself empty for a deal, don't assume the whole approach is broken — spot check a few more deals before concluding the field is unpopulated.

Pull:
- **Rep** Commit / Gut / Best Case (the AE's own numbers, sourced from "PTM")
- **RVP** Commit / Gut / Best Case (the SE/sales manager's numbers)
- **AVP** Commit / Gut / Best Case (leadership's numbers, i.e. Marisa's level)

Add a compact Forecast View table to the one-pager (Commit / Gut / Best Case as rows, Rep / RVP / AVP as columns). This is a good gut-check input on its own — a meaningful gap between what the rep, RVP, and AVP each think a deal is worth (e.g. Rep has it in Best Case while AVP has it at $0 across all three, or vice versa) is itself worth flagging in Gaps & Watch Items, not just displaying as a table.

### One-pager format (per deal, in this order)

1. **Header** — Account name, "New Business" or "Upsell/Existing Business" tag.
2. **Snapshot table** — Owner (AE), SE, Segment, Stage, ACV, Weighted ACV, Probability, Close Date, Days in Stage, Days to Close, and **Use Case** (a short phrase, not a sentence — what they're actually trying to do with Sigma, e.g. "MicroStrategy replacement," "Embedded analytics for external customers," "Multi-tenant platform expansion." Derive this from `Opportunity Why Anything`/`Opportunity Why Sigma`, the MEDPIC payload's decision criteria/pain, and the incumbent/competitor fields — don't just restate the opportunity name).
3. **Forecast View table** — Commit / Gut / Best Case as rows, Rep / RVP / AVP as columns, from the section above.
4. **Gut Check** — a colored one-row callout: **GREEN** (healthy, on track), **YELLOW** (real open risk or dependency, not yet alarming), **RED** (stalled, blocked, or missing fundamentals). One sentence stating why, not a hedge.
5. **Where things stand** — a 3-6 sentence plain-English recap: what this deal is, who's driving it, what's actually happening right now, and what the real blocker or next milestone is. Write like you're catching a colleague up, not filing a report. Pull this from the Next Step history and MEDPIC payload, not just the latest line.
6. **Gaps & Watch Items** — bullets, only for things actually missing or concerning (no economic buyer contact, stale MNDA, blank MEDPICC this late in-stage, competitive price pressure, external/internal blockers outside the rep's control, slipped close dates, account history worth knowing, meaningful Rep/RVP/AVP forecast disagreement). Don't pad with a full MEDPICC checklist if most of it is fine — see `references/risk_rules.md` for the underlying logic, but keep the output to what's real for this deal.
7. **Momentum** (one italic line) — call count, date range, average external sentiment, and anything notable about recent cadence (gone quiet relative to its own history, or the reverse).

Page-break between deals so it reads as a real packet. Don't add a MEDPICC letter-by-letter table here — that level of detail belongs in the single-deal deep dive if she wants to go further on a specific flagged deal.

### Scale notes

- If the qualifying set is very large (~100+), tell her the count before generating, and confirm you're proceeding with the full set rather than assuming — she's asked for the full version before, but scope can change (ACV floor, roster) run to run.
- Building 100 one-pagers means 100 rounds of synthesis — budget for a long single turn. Don't silently truncate the deal list to make it faster; if you need to shorten scope, say so and ask.
- She may ask you to run this on a cadence (weekly, pre-pipeline-review). There's no scheduling mechanism to set up — just note she can ask again any time with the same (or updated) scope.

## Single-deal deep dive

### Step 0 — Connect to Sigma

Call `Sigma MCP:begin_session` once at the start if you haven't already this conversation.

Find the `Sales` data model:
```
Sigma MCP:search { query: "Sales", entityTypes: ["data-model"] }
```
As of this writing it resolves to `dataModelId: 3e8d0574-d4d4-47b4-9995-9cb2b063d077` — but always confirm via search rather than hardcoding, since IDs can differ across environments (staging vs. prod) or change over time.

Then `describe` that data model (`type: "datamodel"`) to confirm the element IDs for:
- `Opportunities Enriched` (known as of writing: `Ftb6toyfCb`)
- `Gong Calls Enriched` (known as of writing: `LdFdB9e_Lk`)

Call `describe` with `type: "datamodel-element"` on each before writing SQL, so you have the real column IDs — they're opaque (e.g. `_BbsT0LpKk`) and do change; never guess them from a previous run.

### Step 1 — Resolve the deal

Query `Opportunities Enriched` filtering on `Opportunity Name` and/or account-related fields with `ILIKE '%<name>%'`. If the user names an account rather than a specific opp, there may be multiple opportunities (new business, renewal, upsell) — list them with stage + ACV and ask which one, unless only one is open.

Pull the full opportunity row for the resolved `Opportunity Guid`, including:
- Identity/ownership: Opportunity Name, Owner (AE) Name/Email, Sales Engineer Name, Solution Architect Name, Sales Segment, Opportunity Type
- Deal mechanics: Stage, ACV Amount, Weighted ACV, Probability, Close Date, Days to Close, Days in Stage, Close Fiscal Quarter
- MEDPICC raw material: `Medpic Payload` (parse this — it's a variant/JSON with Metric, Economic Buyer, Decision Criteria, Paper Process, Identify Pain, Champion), `Opportunity Competitor` + `Opportunity Competitor List Desc`, `Opportunity Why Anything`, `Opportunity Why Buy Now`, `Opportunity Why Sigma`
- Momentum signals: Next Step, Last Gong Call Next Steps, Num Calls, Days Since Last Call, Avg Days Between Calls, Outbound/Inbound Emails, Days Since Last Email
- If closed: Loss Reason, Loss Reason Details, Closed Lost DQ Notes, Product Loss Reason (skip these fields entirely if the deal is open — don't pad the report with empty loss-reason rows)

### Step 2 — Pull the Gong history for that deal

Query `Gong Calls Enriched` filtering `"Opportunity Guid" = '<guid>'`, ordered by call date. Pull per call: Call Title, Call Date, Stage at Time of Call, Internal/External Participant Names, Full/Internal/External Sentiment Scores, Recording Duration, Call URL.

Look at the **trend**, not just the latest number — a sentiment score dropping across the last 2-3 calls is a stronger signal than any single score. If it's useful to ground a specific risk or objection, pull short relevant snippets from `External Speaker Transcript` for the most recent 1-2 calls — this is internal company data, so quoting a prospect's own words in an internal deal report is fine, but keep it to what actually supports a point, not a transcript dump.

### Step 3 — Synthesize into the report

Build a Word document (consult `/mnt/skills/public/docx/SKILL.md` first) with these sections. Keep language plain and direct — no forced enthusiasm, no hedging filler, short declarative sentences, matching a "tell it straight" tone rather than a polished sales-y one.

1. **Deal Snapshot** — name, account, owner/SE/segment, stage, ACV, close date, days in stage, days to close, one line on trajectory (accelerating / stalled / slipping).
2. **Forecast View** — Rep / RVP / AVP Commit-Gut-Best Case table (see "Forecast view" section above) plus a line calling out any meaningful disagreement between the tiers.
3. **MEDPICC Assessment** — a table, one row per MEDPICC letter (Metric, Economic Buyer, Decision Criteria, Decision Process/Paper Process, Identify Pain, Champion, Competition). For each: Status (Confirmed / Partial / Unknown), the evidence from the Medpic Payload or AE's Why-fields, and — if Unknown or Partial — what's missing.
4. **Objections & Sentiment** — call-by-call sentiment trend, named competitors, any specific objections or hesitations surfaced in transcripts, referenced by call date.
5. **Risk Flags** — pull from `references/risk_rules.md`. Only list flags that actually fire for this deal; don't list a checklist of things that are fine.
6. **Recommended Actions** — one concrete action per real risk/gap identified above. No generic filler action items.

See `references/risk_rules.md` for the specific thresholds and MEDPICC-gap logic to apply when deciding what counts as a risk.

## Notes

- If the user wants to run this as a recurring habit (e.g. every Monday before pipeline review), just note that they can ask again any time with the same deal/segment name — there's no need to explain scheduling mechanics unless they ask.
- If `Medpic Payload` or Gong data is empty for a deal, say so plainly in the report rather than guessing or filling gaps with assumptions.

## Known limitation: SE Command Center notes are not pullable

The "SE Command Center" workbook (`workbookId: 343055f5-4ba8-4567-bc37-1cdeabb1c8a3`) has two notes elements that would be genuinely useful here:
- **Forecast Note History** (`3-2TYKH5nN`) — manager/RVP-authored Heat Check + notes (Good 🔥 / Neutral 😐 / Bad 🧊). The underlying query is already filtered server-side to leadership emails only, so this is specifically the manager's-perspective read on a deal.
- **Note History** (`NwMB0b6eVs`) — the mirror element, SE-authored notes (leadership emails explicitly excluded).

Both are backed by a query that includes an `Opportunity GUID` column (confirmed via `list_workbook_queries`), but that column is stripped from what's exposed to `describe`/`query` on these two elements — querying it directly returns "Unresolved column." There's no way found so far to join these notes to a specific opportunity through the MCP connector; the link only works as a live click-through filter in the Sigma UI.

Marisa confirmed (8/26/2026) to skip pulling these notes into the Territory Pulse Report or single-deal deep dive until/unless that GUID becomes queryable. Don't re-attempt this without checking with her first — if you find a workaround (e.g. the workbook sharing changes, or a differently-scoped element exposes the join), confirm with her before building it in, since she deliberately chose to defer this rather than have it silently missing.
