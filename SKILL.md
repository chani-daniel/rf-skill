---
name: rf-component-search
description: Parametric search for RF/microwave components (amplifiers, mixers, filters, switches, attenuators, couplers, etc.) against a multi-parameter spec, with 100%-reliable datasheet verification. Use whenever the user gives a component spec with electrical parameters (frequency range, gain, P1dB, OIP3, NF, insertion loss, isolation...) and wants matching parts — e.g. "מצא לי מגבר", "תחפש רכיב", "צריכה mixer", "find an amplifier 8-12 GHz gain 20dB", or pastes a spec line. Also use when the user asks to check "if there's anything on the market" matching parameters. The user searches AFTER already checking their 12 usual vendor sites, so those are always excluded — see the Vendor Lists section. Trigger even for short spec-only messages with no verb.
---

# RF Component Parametric Search

Find RF/microwave components that **fully match** a multi-parameter spec, and prove it. The user is an RF procurement/engineering professional. This task normally takes her days of manually opening result after result; the value you add is speed **without sacrificing reliability**. A false "match" that fails on one parameter wastes her time and erodes trust — a verified "no match found" is a perfectly good answer.

## Report language

Report in Hebrew. Keep parameter names, part numbers and units in English (Gain, OP1dB, OIP3, NF, GHz, dBm) — that is how RF engineers read them.

## Required reference files

Before Step 1, load both:

- **`rf-parameter-rules.md`** — the general, component-agnostic rules for how any parameter is handled (which parameters may be filtered on, the datasheet-suitability rule, `min`/`max`/`contains` semantics, guaranteed-column-over-typ, mandatory shown conversions).
- **The per-component module matching the requested component type** (e.g. `rf-amplifier-module.md` for amplifiers) — the concrete list of searchable parameters, their units, directions, derivation formulas, fixed conventions, and physics sanity checks.

Only parameters defined in the loaded component module may be used as filters. If no module exists for the requested component type, say so and ask how to proceed rather than improvising parameters.

## Core definitions — defined once, referenced throughout

The rest of the skill refers back to this section instead of re-explaining these. Read it first.

### Verdicts

- **✅ full match** — every required parameter confirmed from the manufacturer datasheet/catalog table.
- **⚠️ borderline** — meets spec but only as typ, or exactly at the limit, or one spec unverifiable, or the access-blocked case below.
- **❌ rejected** — a required parameter fails; state which and by how much.

### site-checkable vs datasheet-only

The loaded component module classifies each parameter as **site-checkable** (reliably shown/filterable on parametric sites and catalog tables) or **datasheet-only** (reliably found only on the datasheet). This classification drives every screen/verify decision below.

### The site-data rule (promote-only, except one gate)

Site data (search snippets, distributor/parametric tables) may **freely add or promote** a candidate, but may **reject** one **only** at the Step 2.7 screen, and only via a catalog fact or a clear miss on a site-checkable parameter the site actually exposes. Site data cannot *confirm* a match (typ-at-one-frequency, missing conditions, plain errors) — which is also why it cannot fail a *borderline* one. Everything not cleanly rejected at 2.7 becomes a **plausible candidate** whose datasheet must be opened. **Never silently skip or drop a source** in any step — an empty/blocked source is logged, not omitted.

### Outcome categories (used in the coverage sheet and audit)

- `checked/no candidates` · `checked/found X` · `not covered`
- **`rejected at site screen`** — dropped at Step 2.7 on a clear miss on a site-exposed parameter (or a catalog fact), logged with the failing site parameter and its value. A definitive rejection; **not** flagged "not datasheet-verified". Not opening the datasheet of a part that already failed on the site is expected, not a gap.
- **`not datasheet-verified`** — passed **every** specified site-checkable parameter at 100% (clear of the guard band) yet the datasheet was not checked (the access-blocked ⚠️ case below). The **only** outcome that carries this flag.
- **`datasheet inaccessible`** — the datasheet could not be fetched; log the alternative sources tried and whether the part ended as a ⚠️ not-verified match or an unverifiable reject.

### Access-blocked datasheet logic (referenced by Steps 3, 3.5, 4, 5)

A blocked datasheet is an *access* failure, never a parameter failure. First **exhaust alternative datasheet sources, logging each one tried**: everything.rf's linked/mirrored datasheet, a search-engine cached copy, a distributor-hosted PDF, and the `resources.ampheo.com/static/datasheets/<vendor>/<part>.pdf` mirror pattern. Only if **all** fail, classify by the module:

- **Every** specified parameter is site-checkable, was actually shown on a site, and clears the spec **beyond** its guard band (merely inside the guard band ≠ clear) → **⚠️ match, `not datasheet-verified`**. Sits in the matches sheet with the flag "לא אומת — גישה ל-datasheet חסומה" and an aggregator link in place of the unreachable datasheet. **Never** shown as ✅. This is the one case where site data may seat a part in the matches sheet.
- **Any** specified parameter is datasheet-only, or is site-checkable but was never shown on a site, or passed only *within* its guard band → **unverifiable**. Goes to the rejected sheet, labelled distinctly "לא נפסל על פרמטר — נדרש אימות ב-datasheet שלא היה נגיש; כדאי לשקול פנייה ליצרן" — never conflated with a real parameter failure.

## Workflow

### Step 1 — Parse the spec, then clarify ONCE before searching

Extract every parameter into a requirements table: name, value, direction (min/max/range), and whether it's hard or soft. Specs arrive as terse free text ("pidb 20 min, OIP3 30dbm, NF 6db max").

Ambiguities are the #1 source of wrong results. Before searching, ask the user (in one round, using the question tool if available) only about what is genuinely ambiguous:

- **Frequency**: must the part's band merely contain the requested range (usual), or match it exactly?
- **Two-sided ranges** (e.g. "gain 20-30dB"): is the upper bound hard (a part with 33dB is rejected) or advisory?
- **min/typ/max**: do required values need to be guaranteed (min/max columns) or is typical acceptable?
- **Form factor**: MMIC/SMT, connectorized module, bare die — or anything?

Do NOT re-ask things already answered in the conversation, and don't ask about parameters that are already unambiguous. Known user defaults: band containment is fine; typical values acceptable; any form factor.

### Step 2 — Search wide (candidates), never trusting search snippets

Discovery must not hang on any single list of vendors. A small vendor that a hand-maintained list omits **and** Google indexes poorly falls through both nets and is missed entirely — this is exactly how Aelius Semiconductors' ASL 4020 / ASL4065 were missed on a real query ("amplifier 14–15 GHz, Gain≥20 dB, P1dB≥24 dBm"), even though everything.rf lists Aelius with 111 amplifiers. So run **three independent discovery paths** and pool everything they surface into one candidate queue. **Dedupe the queue** (same vendor/part surfaced by more than one path is processed once) before it flows into the Step 2.7 screen. A vendor is missed only if **all three** paths miss it — and even then the report says so honestly (Step 4).

These paths are component-agnostic: the category and category hints vary by component type, the mechanism does not. In every path, **always exclude the user's 12 vendor domains** using blocked_domains (Vendor Lists section). Per the site-data rule, a hit from any path is only a *candidate* until Step 3.

**Path A — Parametric aggregator query (operate the filters; do NOT keyword-search).**
On everything.rf — the best RF-specific parametric DB — and, where the component type fits, Mouser and Digi-Key parametric search, drive the site's *own parametric engine*: pick the component category for **this** query (Amplifiers, Mixers, Filters, LNAs, …), set the numeric filters from the current spec (whichever parameters the loaded module marks site-checkable), and **page through ALL result pages**. This is a *category-wide* sweep filtered by the current spec — never a search for a pre-named part; return every part the filters admit. A keyword web-search against these domains is **NOT** a substitute for their parametric engine: it reproduces the indexing weakness that caused the Aelius miss above.

**Path B — Part-graph traversal (vendor DISCOVERY through parts, not names).**
One good candidate is a thread to pull; its purpose is to surface **new vendors no list or directory contains** — found through parts, not names. From every confirmed or plausible candidate, run derived searches:

- "alternatives to <part number>" / "<part number> equivalent" / "cross reference"
- sibling parts in the same family (a vendor with an ARF1211 likely has ARF12xx neighbors — check the vendor's category page)
- each aggregator's "similar products" section on the candidate's page
- re-run the best query in the found parts' vocabulary ("driver amplifier" vs "gain block" vs "medium power amplifier").

Whenever a competing part comes from a **vendor not yet seen this session**, add that vendor and sweep its catalog for the category. **Loop until a wave surfaces no new vendors and no new plausible candidates, up to a ceiling of 3 waves** (or stop earlier once a wave adds nothing) — then continue; note the ceiling in the coverage statement if it was hit.

**Path C — Cache sweep.**
Sweep the manufacturers in the vendor cache (Vendor Lists) relevant to the component category, checking their catalogs directly (site search or catalog PDF fetch). The cache is **not** the definition of which vendors exist — Paths A and B are; the cache only makes access to a *known* vendor cheap. If a catalog PDF returns empty, it is probably scanned/image or bot-blocked — say so and try the vendor's HTML pages or everything.rf's copy instead (per the site-data rule: never silently skip).

**Grow the cache.** Whenever Path A or Path B surfaces a vendor not already in the cache, append it there with the access metadata learned (domain, catalog URL, parse/access notes, component categories). The cache grows automatically toward completeness for the categories searched; no human maintains it.

### Step 2.7 — Site-level pre-screen (Stage 1: cheap filter before any PDF)

Datasheet fetches are the expensive part. Before opening any datasheet, screen candidates using only the data the sites expose — no PDF. The screen uses only the *site-checkable* parameters the query actually specified (see `rf-parameter-rules.md` for mechanics and tolerance):

1. On each site, see which of those query parameters it can filter on or shows in its table.
2. Compare each to the query, using the semantics/tolerance in `rf-parameter-rules.md`.
3. **Passes** when every site-checkable query parameter the site exposes matches → promoted to Step 3.
4. **Rejected at Stage 1** only when a site-exposed parameter clearly fails or a catalog fact rules it out (band that cannot contain the request, wrong type/form factor, excluded vendor).

Safety rules (all generic in `rf-parameter-rules.md`): parameters the user did not specify or not in the module are ignored (treated as a match); a datasheet-only parameter is never screened here; a site-checkable parameter the site does not expose cannot reject the part (promote, confirm at datasheet); **when in doubt, promote** — Stage 2 catches borderline parts.

**Log every Stage-1 reject** as `rejected at site screen` (see Outcome categories) — the clean-rejection case, **not** "not datasheet-verified".

### Step 3 — Verify every candidate against primary sources

A candidate becomes a match only after every required parameter is confirmed from the **manufacturer's datasheet or catalog table** (fetch the actual PDF/product page). Distributor summaries and snippets routinely show typ at one frequency point, omit conditions, or are wrong — see the site-data rule: a part is rejected here only against an actual datasheet value; site data may only have promoted it in.

**Efficiency:** verify only the parameters still open. A site-checkable parameter that already cleared the spec **beyond its guard band** at Step 2.7 need not be re-read from the datasheet — focus the fetch on datasheet-only and borderline parameters.

For each parameter record: the actual value, whether it is min/typ/max, and any conditions (temperature, frequency point vs full band). Watch for:

- Specs guaranteed only at +25°C vs over temperature — note which.
- NF/gain at a single frequency vs across the band. The requested band must be inside the datasheet's *specified* range, not just the "operating" range.
- Column-header typos (a "Min/Typ" header on a Noise Figure column almost certainly means Max/Typ — flag, don't assume).
- Parameters listed TBD or absent → **unverifiable**, not a match. Say "requires manufacturer contact", never guess.

Record the **margin** per parameter (e.g. OIP3 required ≥30, actual +37 → margin +7dB) so the user can rank matches. Assign a verdict (see Verdicts).

**If the datasheet cannot be fetched** (bot-block / 409 / 404 / not indexed): follow the Access-blocked datasheet logic in Core definitions.

### Step 3.5 — Independent re-verification (the trust layer)

Reading errors and confirmation bias are real. Before reporting, re-verify **✅/⚠️ candidates** from scratch. To save tokens, **prioritize the borderline ones** (small margin, typ-only, single-reading) — a parameter that cleared the spec by a wide margin is low-risk; a comfortably-clear ✅ with quoted values need not be re-fetched. First determine which mode is available — subagent capability depends on the *runtime environment*, not the model; do not assume it is present, and state the mode used in the coverage statement.

**Preferred — subagent mode (whenever available):** spawn a verification agent that receives ONLY the requirements table and the candidate part numbers + datasheet URLs — **not** your conclusions, values, or verdicts — and independently extracts each parameter into its own table. Batch the candidates into **one** agent rather than one per part. Compare its table to yours; any discrepancy → fetch the datasheet again and resolve explicitly. An agent that never saw your answer cannot rubber-stamp it.

**Fallback — hardened single-agent mode (when subagents are unavailable — likely common, including a strong model in plain chat):** a plain "read it again" is weak because you remember your conclusion. Make the re-read mechanical:

1. Do NOT look at your first table. Re-fetch each prioritized datasheet fresh and extract values into a **blank** second table, quoting for every value the exact datasheet string + location (page / table / row, e.g. *"Gain 22 dB typ, p.3 'Electrical Specifications' table, 8–12 GHz row"*). A value with no locatable quote is **unverifiable** → downgrade it.
2. Only then diff against the first table. Any mismatch → open the datasheet a third time; quoted source wins over memory.
3. Because this mode is self-checking, treat its confidence as lower: any ✅ resting on a single ambiguous reading becomes ⚠️, and the coverage statement notes single-agent re-verification.

Sanity checks in **either** mode (violations = red flag, re-check the source): apply the loaded module's physics checks; a distributor page disagreeing with the datasheet loses (datasheet wins; note the discrepancy).

Only candidates that survive re-verification are reported. **Exception — a ⚠️ access-blocked match has no datasheet to re-fetch:** re-verify against the *site* values (re-run the parametric query fresh, confirm each value still clears the spec beyond its guard band) and keep it ⚠️ `not datasheet-verified`; name it in the coverage statement as a candidate that could not be datasheet-verified.

### Step 4 — Report: chat table + Excel

**Chat**: one table of matches/borderlines (part number, vendor, band, each required parameter with min/typ noted, verdict), followed by a short "checked and rejected" list — part, failing parameter, actual value. Placement follows the Outcome categories: a ⚠️ access-blocked match belongs in the matches table with its "לא אומת — גישה ל-datasheet חסומה" flag (never in the rejected list); an access-blocked *unverifiable* part goes in the rejected list marked "לא נפסל על פרמטר — כדאי לשקול פנייה ליצרן". The rejected list is what convinces the user the search was real.

**Excel** (use the xlsx skill; RTL Hebrew — follow the hebrew-office-documents skill) — **three sheets, all mandatory, even with one or zero matches**:

1. **התאמות** — matches with all parameters and datasheet links, including ⚠️ `not datasheet-verified` rows (flag "לא אומת — גישה ל-datasheet חסומה"; aggregator link in place of the unreachable datasheet).
2. **נבדקו ונפסלו** — every candidate that reached datasheet-check and failed, with the failing parameter and actual value. Also holds access-blocked *unverifiable* parts, labelled distinctly "לא נפסל על פרמטר — נדרש אימות ב-datasheet שלא היה נגיש; כדאי לשקול פנייה ליצרן" — not as a parameter failure.
3. **יומן כיסוי** — a **paths table** (each of the three discovery paths with its outcome, e.g. "everything.rf parametric ✓ — N pages"; "part-graph traversal ✓ — M waves, K new vendors"; "cache sweep ✓") **plus a manufacturers table** (below). Also record the re-verification mode used (subagent / single-agent).

   The manufacturers table rules:
   - **Exhaustive** — one row for every vendor touched at least once through any path, identical format for all, regardless of outcome. A vendor that returned nothing is listed `checked/no candidates` — **never omitted**. Proving what was searched is the point; searched-and-empty is as important as a match.
   - **Aggregators expanded, never collapsed** — when Path A drives a parametric engine, add **one row per vendor that engine surfaced** behind the query, each in the same format. A "everything.rf — 426 amps" single row is not acceptable. The aggregator may *additionally* keep one summary row in the paths table. A vendor surfaced only inside an aggregator gets the outcome the aggregator data gives it (`checked/found X`, `rejected at site screen`, …).
   - **"שאילתה שנשלחה" (query sent) column, mandatory** — the actual query sent to that source: the exact keyword/web-search string; for a parametric engine the filter set applied (e.g. "Amplifiers; 14–15 GHz; Gain≥20 dB; P1dB≥24 dBm"); for a catalog/PDF sweep the site-search term or catalog URL. Multiple queries → newline-separated in the cell or one row per query. A `not covered` source leaves it empty.
   - Outcomes use the Core Outcome categories.

Sheets 2–3 are not decoration — they ARE the product. A report with one match and no rejection/coverage record is unverifiable: the user cannot tell a thorough search from a lucky hit, and will re-check everything manually. An empty sheet with just a header ("אף מועמד לא נפסל בשלב datasheet") is itself meaningful.

**One-match warning**: 0–1 matches after a real sweep is possible but suspicious. Before accepting, run at least one more Path B wave and confirm all three paths (A/B/C) ran for this category — then report, saying explicitly in the coverage statement that this was done.

**End with an honest coverage statement**: which of the three paths ran and their outcomes; which sources were fully swept vs only sampled; whether "no match" means "none exists" or "none found in sources covered"; and the re-verification mode. Coverage is bounded by what these sources contain — state plainly that a vendor absent from **every** path may still be missed; keep the residual gap visible, never disguised as a clean "no match". If nothing matches, show the nearest misses with their exact gaps (a 2dB gap is actionable — she may relax the spec).

### Step 5 — Final audit before sending

Run this checklist; fix anything that fails before reporting. (Each item points back to the rule it enforces.)

- [ ] Every ✅/⚠️ part: every required parameter has an actual value, min/typ/max label, margin, and a working datasheet/catalog link — **or**, for a ⚠️ access-blocked match, site values with guard-band-clear margins, an aggregator link, and the "not verified" flag.
- [ ] Every ✅/⚠️ part survived Step 3.5 re-verification; the mode (subagent / single-agent) is stated in the coverage statement.
- [ ] Every datasheet ❌ rests on an actual datasheet value — never on snippet evidence. The only non-datasheet entries in the rejected sheet are `rejected at site screen` and access-blocked *unverifiable* parts, each labelled per the Outcome categories.
- [ ] `rejected at site screen` vs `not datasheet-verified` used exactly per the Outcome categories (not conflated).
- [ ] Any inaccessible datasheet followed the Access-blocked logic: alternatives tried and logged, then classified as ⚠️ not-verified match (התאמות) or unverifiable reject (נבדקו ונפסלו). Never auto-dumped into rejected, never shown as ✅.
- [ ] Every ❌ part names the specific failing parameter and its actual value (except access-blocked unverifiable, which names the datasheet-only parameter it couldn't check and flags "consider contacting the manufacturer").
- [ ] No part is from an excluded vendor (including legacy brands now on excluded domains).
- [ ] All three paths ran; the coverage statement reports each outcome (A pages paged; B waves + new vendors, and whether the 3-wave ceiling was hit; C cache sweep). A skipped path is named, not silently omitted.
- [ ] The coverage statement states plainly that coverage is bounded by these sources and a vendor absent from every path may still be missed.
- [ ] Any vendor newly discovered by Path A/B was appended to the vendor cache with access metadata.
- [ ] The Step 1 clarifications are reflected (e.g. a hard bound → no match exceeds it).
- [ ] The module's sanity checks were applied; any violation re-checked against the source.
- [ ] Numbers in the chat table match the Excel exactly.
- [ ] The Excel has all three sheets (התאמות / נבדקו ונפסלו / יומן כיסוי) — even if a sheet is header-only.
- [ ] The יומן כיסוי manufacturers table: has the **"שאילתה שנשלחה"** column with the real query per checked source; is **exhaustive** (every touched vendor, including empty ones); and every Path-A aggregator is **expanded** into per-vendor rows.
- [ ] If 0–1 matches: an extra Path B wave was run and the coverage statement says so.

## Efficiency notes

- Two-stage search — cheap wide search → **Step 2.7 site screen** → datasheet fetches for survivors only — is the main token saver. Datasheets open only for parts that pass the screen.
- **Dedupe the candidate queue** (Step 2) before Step 2.7 so a vendor/part surfaced by multiple paths is processed once.
- At Step 3, don't re-read from the datasheet any site-checkable parameter that already cleared the spec beyond its guard band; fetch for datasheet-only and borderline parameters.
- At Step 3.5, prioritize re-verification on borderline candidates and batch subagent verification into one agent.
- If the user names a vendor missing from the cache, add it (Path C) for the session with whatever access metadata you learn — same auto-append rule Paths A/B use.

## Vendor Lists

### ALWAYS EXCLUDE — the user's 12 pre-checked sites

The user checks these manually before every search. Pass all of them in blocked_domains on every web search, and never present parts from them:

| Vendor | Domain |
|---|---|
| Mini-Circuits | minicircuits.com |
| Qorvo (incl. Custom MMIC, Sirenza legacy parts) | qorvo.com |
| MACOM (incl. Mimix legacy) | macom.com |
| Analog Devices (incl. Hittite legacy) | analog.com |
| UMS | ums-rf.com |
| 3R Waves | 3rwave.com |
| AMCOM | amcomusa.com |
| VectraWave | vectrawave.com |
| Guerrilla RF | guerrilla-rf.com |
| Microchip | microchip.com |
| Marki Microwave | markimicrowave.com |
| RW MMIC | rwmmic.com |

Legacy-brand note: parts whose datasheets now live on an excluded domain (e.g. Custom MMIC → qorvo.com, Hittite → analog.com) count as excluded.

### VENDOR CACHE — access knowledge for known manufacturers

This is a **cache**, not the source of truth for which vendors exist. The universe of vendors is defined by Path A (parametric aggregators) and Path B (part-graph traversal); this list's only job is to make access to an *already-known* vendor cheap and reliable — per-vendor access metadata (domain, catalog URL, parse/access notes, category hints). Path C sweeps it. Generic web search misses many of these (poor indexing), so checking their catalogs directly still matters — but absence here no longer means a vendor won't be found.

The cache is **self-growing** — no human maintains it:

- Whenever Path A or Path B discovers a vendor NOT listed here, append it with the access metadata learned (domain, catalog URL, parse/access notes, component categories).
- The cache MAY be seeded or refreshed from **everything.rf's Companies directory** for the relevant category.
- Not every vendor is relevant to every part type — use the category hints.

**MMIC / semiconductor:**
- Altum RF — altumrf.com (amps, X/Ku-band; also stocked at rellpower.com)
- BeRex — berex.com (amps, LNAs)
- CEL (California Eastern Labs) — cel.com (LNAs, discrete)
- Skyworks — skyworksinc.com (amps < ~6 GHz, switches, mixers)
- NXP — nxp.com (power, drivers)
- Wolfspeed — wolfspeed.com (GaN power)
- Ampleon — ampleon.com (power)
- Broadcom/Avago legacy MMICs — broadcom.com
- Mercury Systems / Atlanta Micro — mrcy.com (AM-series amps)

**Connectorized modules / hybrid amplifiers:**
- Ciao Wireless — ciaowireless.com (very broad amp catalog; catalog PDFs parse well)
- Narda-MITEQ (L3Harris) — nardamiteq.com (huge AMF amplifier catalog — poorly indexed, sweep directly)
- Erzia — erzia.com
- B&Z Technologies — bnztech.com
- Planar Monolithics Industries (PMI) — pmi-rf.com
- Cernex / CernexWave — cernex.com
- Wenteq Microwave — wenteq.com (store.wenteq.com has per-part pages)
- Pasternack — pasternack.com (site bot-blocked; datasheets mirrored at resources.ampheo.com/static/datasheets/pasternack/<part>.pdf)
- Fairview Microwave — fairviewmicrowave.com
- Lotus Communication Systems — lotussys.com
- AML — amlj.com
- Elite RF — eliterfllc.com
- Triad RF — triadrf.com
- Spacek Labs — spaceklabs.com (mm-wave)
- Quantic brands (X-Microwave, PMI, Corry...) — catalog.xmicrowave.com

**Parametric search engines / distributors (search these too, not just Google):**
- everything.rf — best RF-specific parametric DB; also mirrors specs of poorly-indexed vendors
- Mouser, Digi-Key — parametric filters for SMT/MMIC parts
- RFMW — rfmw.com — RF-specialist distributor
- Richardson Electronics — rellpower.com (Altum RF and others)

This cache is the accumulated institutional knowledge of the search — Paths A and B grow it automatically over time.
