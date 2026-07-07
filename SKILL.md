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

Discovery must not hang on any single list of vendors. A small vendor that a hand-maintained list omits **and** Google indexes poorly falls through both nets and is missed entirely — this is exactly how Aelius Semiconductors' ASL 4020 / ASL4065 were missed on a real query ("amplifier 14–15 GHz, Gain≥20 dB, P1dB≥24 dBm"), even though everything.rf lists Aelius with 111 amplifiers. So run **three independent discovery paths** and pool everything they surface into one candidate queue, which then flows unchanged into the Step 2.7 site screen. The paths are named so the audit can report on each (Step 5). A vendor is missed only if **all three** paths miss it — and even then the report says so honestly (Step 4), never disguising the gap as a clean "no match".

These paths are component-agnostic: the category and category hints vary by component type, the mechanism does not. In every path, **always exclude the user's 12 vendor domains** using blocked_domains — full list in the Vendor Lists section below. She already checked them; results from them are pure noise. And a hit from any path is only a *candidate* until Step 3 — never trust a search snippet as truth.

**Path A — Parametric aggregator query (operate the filters; do NOT keyword-search).**
On everything.rf — the best RF-specific parametric DB — and, where the component type fits, Mouser and Digi-Key parametric search, drive the site's *own parametric engine*: pick the component category for **this** query (Amplifiers, Mixers, Filters, LNAs, …), set the numeric filters from **whatever spec the user ran this session** (frequency band, gain floor, P1dB floor — whichever parameters the loaded module marks site-checkable), and **page through ALL result pages**, not just page one. This is a *category-wide* sweep filtered by the current spec — you are never searching for a pre-named part; you return every part the filters admit. A keyword web-search against these domains is **NOT** a substitute for their parametric engine: it reproduces the very indexing weakness the aggregator exists to bypass. That weakness is what caused the past miss — a valid vendor that sits inside the aggregator's category once the spec's filters are set, yet is invisible to a generic Google query. This path gives broad cross-vendor recall in one query and surfaces vendors you never knew to look for.

**Path B — Part-graph traversal (vendor DISCOVERY through parts, not names).**
One good candidate is a thread to pull, and its explicit purpose here is to surface **new vendors that no list or directory contains** — because it finds them through parts, not vendor names. From every confirmed or plausible candidate, run derived searches:

- "alternatives to <part number>" / "<part number> equivalent" / "cross reference"
- sibling parts in the same family (a vendor with an ARF1211 likely has ARF12xx neighbors — check the vendor's category page)
- each aggregator's "similar products" section on the candidate's page
- re-run the best query in the found parts' vocabulary (vendors describe the same thing differently: "driver amplifier" vs "gain block" vs "medium power amplifier").

Whenever a competing part comes from a **vendor not yet seen this session**, add that vendor and sweep its catalog for the category. **Loop until a wave surfaces no new vendors and no new plausible candidates.** This is the path that catches vendors absent from every aggregator directory, precisely because it discovers them via parts.

**Path C — Cache sweep.**
Sweep the manufacturers in the vendor cache (formerly the "sweep list" — see Vendor Lists) relevant to the component category, checking their catalogs directly (site search or catalog PDF fetch). The cache is **not** the definition of which vendors exist — Paths A and B are; the cache only makes access to a *known* vendor cheap and reliable.

If a catalog PDF fetch returns empty, it is probably a scanned/image PDF or bot-blocked — say so and try the vendor's HTML product pages or everything.rf's copy instead. **Never silently skip a source**, in any path.

**Grow the cache.** Whenever Path A or Path B surfaces a vendor not already in the cache, append it there with whatever access metadata was learned (domain, catalog URL, parse/access notes, component categories) — see the Vendor Lists section. The cache grows automatically toward completeness for the categories actually searched; no human maintains it.

### Step 2.7 — Site-level pre-screen (Stage 1: cheap filter before any PDF)

Datasheet fetches are the expensive part. Before opening any datasheet, screen candidates using only the data the sites themselves expose — no PDF. Only candidates that **survive** this screen advance to Step 3.

The loaded component module classifies each of its parameters as **site-checkable** (reliably shown or filterable on parametric sites and catalog tables) or **datasheet-only** (reliably found only on the datasheet). The screen uses only the *site-checkable* parameters that the user's query actually specified — see `rf-parameter-rules.md` for the full mechanics and matching tolerance:

1. On each site you enter, see which of those query parameters that site can actually filter on or shows in its table.
2. Compare each such parameter to the query, using the matching semantics and tolerance in `rf-parameter-rules.md`.
3. A candidate **passes the screen** when every site-checkable query parameter the site exposes matches. It is then promoted to Step 3 for full datasheet verification of the remaining parameters.
4. A candidate is **rejected at Stage 1** only when a site-exposed parameter clearly fails, or a catalog fact rules it out (a listed band that cannot contain the request, wrong component type/form factor, or an excluded vendor).

Rules that keep the screen safe (all defined generically in `rf-parameter-rules.md`):

- **Parameters the user did not specify, or that are not in the module, are ignored** — treated as a match, never a reason to reject.
- **A datasheet-only parameter is never screened here** — it waits for Step 3.
- **A site-checkable parameter the current site does not expose cannot reject the part** — that site simply can't screen it; the part is promoted and the parameter is confirmed at the datasheet.
- **When in doubt, promote.** Stage 2 exists to catch borderline parts.

**Log every Stage-1 reject** in the coverage sheet as *rejected at site screen*, with the failing site parameter and its value. This is a clean rejection on a real discrepancy — do **not** flag it "not datasheet-verified": not opening the datasheet of a part that already failed on the site is expected, not a gap. The *not datasheet-verified* label is reserved for the opposite case — a part that passed **every** specified site-checkable parameter at 100% (clear of the guard band) but whose datasheet could not be checked (the access-blocked ⚠️ match of Step 3). This keeps the screen auditable and a thorough search distinguishable from one that pruned cheaply. Never silently drop a part.

### Step 3 — Verify every candidate against primary sources

A candidate becomes a "match" only after every required parameter is confirmed from the **manufacturer's datasheet or catalog table** (fetch the actual PDF/product page). Distributor summaries and search snippets routinely show typ values at one frequency point, omit conditions, or are simply wrong.

**Site data may reject only through the Step 2.7 screen; otherwise it may promote, never reject.** A search snippet or distributor/parametric-table value can freely *add* a candidate to the datasheet queue. It may *remove* one only at Step 2.7, and only as that step allows: a catalog fact, or a clear miss on a site-checkable parameter the site actually exposes. Every other site signal only promotes — if a parameter is datasheet-only, not exposed on the site, borderline within tolerance, or simply absent, the part is a **plausible candidate** and its datasheet must be opened. The reasons site data can't *confirm* a match (typ-at-one-frequency, missing conditions, plain errors) are why it can't be trusted to fail a *borderline* one. Every Stage-1 reject is logged as *rejected at site screen*, with its failing site parameter and value — not "not datasheet-verified". A part is otherwise rejected only against an actual datasheet value.

For each parameter record: the actual value, whether it is min/typ/max, and any conditions (temperature, frequency point vs full band). Watch for:

- Specs guaranteed only at +25°C vs over temperature — note which.
- NF/gain specified at a single frequency vs across the band. The requested band must be inside the datasheet's specified range, not just the "operating" range.
- Column-header typos in catalogs (a "Min/Typ" header on a Noise Figure column almost certainly means Max/Typ — flag it rather than assume).
- Parameters listed as TBD or absent → the part is **unverifiable**, not a match. Say "requires manufacturer contact", never guess.

Record the **margin** per parameter (e.g. OIP3 required ≥30, actual +37 → margin +7dB). Margins let the user rank matches and see instantly which are comfortable vs marginal.

Verdicts: ✅ full match · ⚠️ borderline (meets spec but only as typ, or exactly at the limit, or one spec unverifiable) · ❌ rejected (state exactly which parameter fails and by how much).

**When the manufacturer datasheet cannot be fetched (bot-block / 409 / 404 / not indexed).** A blocked datasheet is an *access* failure, not a parameter failure — it must never be reported like a part that failed on a spec. First **exhaust alternative datasheet sources** and log each one tried: everything.rf's own linked/mirrored datasheet, a search-engine cached copy, a distributor-hosted PDF, and the `resources.ampheo.com/static/datasheets/<vendor>/<part>.pdf` mirror pattern. Only if **all** of them fail is the datasheet "inaccessible", and then decide by the module's site-checkable / datasheet-only classification of the parameters the user actually specified:

- **Every** specified parameter is site-checkable, was actually shown on a site, and **clearly** satisfies the spec — meaning the site value clears the requirement by a margin *beyond* that parameter's guard band (a value merely inside the guard band is borderline, not a clear pass): report it as a **⚠️ match, not datasheet-verified**. It goes in the matches table/sheet with an explicit "site values only; datasheet access-blocked, not verified" note. It is **never** shown as ✅. This is the one case where site data may seat a part in the matches sheet — permitted only because every required parameter is site-checkable and clears the spec cleanly, and only ever as ⚠️ with the access caveat visible.
- **Any** specified parameter is datasheet-only, or is site-checkable but was never shown on a site, or only passed *within* its guard band: that parameter is genuinely unknown with the datasheet blocked, so the part is **unverifiable**. Send it to the rejected sheet, but labelled distinctly — *"not rejected on a parameter — <param> is checkable only on the datasheet and the datasheet was inaccessible; worth considering a manufacturer/rep inquiry"* — never conflated with a real parameter failure.

### Step 3.5 — Independent re-verification (the trust layer)

Reading errors and confirmation bias are real: once a part "looks like a match", it is easy to misread a column. So before reporting, re-verify every ✅/⚠️ candidate **from scratch**. First determine which re-verification mode is available — subagent capability depends on the *runtime environment*, not the model: the same model may have it in an agentic/orchestration context and lack it in a plain chat interface. Do not assume it is present. State in the coverage statement which mode was used.

**Preferred — subagent mode (use whenever subagents are available):** spawn a verification agent that receives ONLY the requirements table and the list of candidate part numbers + datasheet URLs — **not** your conclusions, values, or verdicts — and independently extracts each parameter value into its own table. Compare its table to yours; any discrepancy → fetch the datasheet again and resolve explicitly. The separation is what makes this trustworthy: an agent that never saw your answer cannot rubber-stamp it.

**Fallback — hardened single-agent mode (when subagents are unavailable — likely the common case, including a strong model in a plain chat run):** a plain "read it again" is weak, because you remember what you concluded. So make the re-read mechanical and hard to fake:

1. Do NOT look at your first table while re-verifying. Re-fetch each datasheet in a fresh read and extract the values into a **blank** second table, quoting for every value the exact datasheet string plus its location (page / table / row heading, e.g. *"Gain 22 dB typ, p.3 'Electrical Specifications' table, 8–12 GHz row"*). A value with no locatable quote is **unverifiable**, not confirmed — downgrade it.
2. Only after the second table is complete, diff it against the first. Any mismatch → open the datasheet a third time and resolve explicitly; the quoted source wins over memory.
3. Because this mode is self-checking rather than independent, treat its confidence as lower: any ✅ that rests on a single ambiguous reading becomes ⚠️, and the coverage statement notes that re-verification was single-agent.

Sanity checks during re-verification, in **either** mode (violations = red flag, re-check the source):

- Apply the loaded component module's sanity checks (each module defines the physics plausibility checks for its component type).
- Distributor page disagrees with datasheet → the datasheet wins; note the discrepancy.

Only candidates that survive re-verification are reported as matches. **Exception — a ⚠️ access-blocked match (Step 3) has no reachable datasheet to re-fetch:** re-verify it against the *site* values instead — re-run the parametric query in a fresh look and confirm each value still clears the spec beyond its guard band — and keep it ⚠️ "not datasheet-verified". The coverage statement must name it as a candidate that could not be datasheet-verified.

### Step 4 — Report: chat table + Excel

**Chat**: one table of matches/borderlines (part number, vendor, band, each required parameter with min/typ noted, verdict), followed by a short "checked and rejected" list — part, failing parameter, actual value. A ⚠️ access-blocked match (site values only, datasheet unreachable) belongs in the matches table with its "לא אומת — גישה ל-datasheet חסומה" flag, never hidden in the rejected list. An access-blocked *unverifiable* part (a datasheet-only parameter that stayed unknown) goes in the rejected list but marked "לא נפסל על פרמטר — כדאי לשקול פנייה ליצרן". The rejected list is what convinces the user the search was real, and saves her re-checking those parts.

**Excel** (use the xlsx skill; RTL Hebrew — follow the hebrew-office-documents skill) — **three sheets, all mandatory, even when there is only one match or zero matches**:

1. **התאמות** — matches with all parameters and datasheet links. Includes ⚠️ *not datasheet-verified* rows (site values cleared the spec but the datasheet was access-blocked): every such row carries the flag "לא אומת — גישה ל-datasheet חסומה" and links to the aggregator page in place of the (unreachable) datasheet.
2. **נבדקו ונפסלו** — every candidate that reached datasheet-check and failed, with the failing parameter and actual value. Also holds access-blocked *unverifiable* parts (datasheet unreachable **and** a required parameter is datasheet-only, so it could not be confirmed) — these are labelled distinctly "לא נפסל על פרמטר — נדרש אימות ב-datasheet שלא היה נגיש; כדאי לשקול פנייה ליצרן", not as a parameter failure.
3. **יומן כיסוי** — a **paths table** (each of the three discovery **paths** with its outcome, e.g. "everything.rf parametric ✓ — N pages"; "part-graph traversal ✓ — M waves, K new vendors found"; "cache sweep ✓") **plus a manufacturers table** listing every vendor/source touched, with its outcome.

   **The manufacturers table must be exhaustive — one row for every vendor the search touched at least once through any path, in the identical format for all of them, regardless of outcome.** A vendor that was checked and returned nothing is listed with outcome `checked/no candidates` — it is **never omitted**. The point of the table is to prove what was searched, so a searched-and-empty vendor is exactly as important as a vendor that produced a match. Do not drop a vendor because it "found nothing".

   **Aggregators must be expanded, never collapsed into a single row.** When Path A drives a parametric engine (everything.rf, Mouser, Digi-Key), the manufacturers table must contain **one row per vendor that engine surfaced** behind the query — every manufacturer that appeared in the filtered result set, each in the same format as a directly-checked vendor. It is **not** acceptable to represent a parametric sweep as a single "everything.rf — 426 amps" row and hide the vendors inside it. The aggregator may **additionally** keep one summary row in the paths table (e.g. "everything.rf parametric ✓ — N pages"), but every individual vendor it returned still gets its own manufacturer row. A vendor surfaced only inside an aggregator and not otherwise checked is listed with the outcome the aggregator data gives it (`checked/found X`, `rejected at site screen`, etc.) and its query column shows the aggregator filter set that surfaced it.

   The manufacturers table **must** include a **"שאילתה שנשלחה"** (query sent) column — the actual query string sent to that site for the search: the exact keyword/web-search string, or for a parametric engine (Path A) the filter set that was applied (e.g. category + each numeric filter and its value: "Amplifiers; 14–15 GHz; Gain≥20 dB; P1dB≥24 dBm"), and for a catalog/PDF sweep the site-search term or catalog URL fetched. One row per vendor/source shows exactly what was asked of it, so the search is reproducible. If a source was reached through several distinct queries, list them (newline-separated in the cell, or one row per query). A source recorded as *not covered* has no query — leave the cell empty. Use these outcome categories: checked/no candidates · checked/found X · not covered · **rejected at site screen** — a candidate dropped at the Step 2.7 site screen on a clear miss on a site-exposed parameter (or a catalog fact), logged with the failing parameter and its site value; a definitive rejection, **not** flagged "not datasheet-verified" · **not datasheet-verified** — a candidate that passed **every** specified site-checkable parameter at 100% (clear of the guard band) yet whose datasheet was not checked (the access-blocked ⚠️ case); the only outcome that carries this flag · **datasheet inaccessible** — the datasheet could not be fetched; list the alternative sources tried (everything.rf mirror, cache, distributor PDF, ampheo mirror) and whether the part ended as a ⚠️ not-verified match or an unverifiable reject. Also record the re-verification mode used (subagent / single-agent).

Sheets 2–3 are not decoration — they ARE the product. A report with one match and no rejection/coverage record is unverifiable: the user cannot tell a thorough search from a lucky first hit, and she will re-check everything manually, losing the entire value of the skill. An empty sheet with a header row ("אף מועמד לא נפסל בשלב datasheet") is itself meaningful information.

**One-match warning**: finding only 0–1 matches after a real sweep is possible but suspicious. Before accepting it, go back to **Path B** and run at least one more part-graph/derived-search wave, and confirm all three discovery paths (A/B/C) actually ran for this category. Only then report, and say explicitly in the coverage statement that this was done.

**End with an honest coverage statement**: which of the three discovery **paths** ran and their outcomes (e.g. "everything.rf parametric ✓ — N pages; part-graph traversal ✓ — M waves, K new vendors found; cache sweep ✓"), which sources were fully swept vs only sampled, whether "no match" means "none exists" or "none found in the sources covered", and which re-verification mode (subagent / single-agent) was used. Coverage is bounded by what these sources contain: state plainly that a vendor absent from **every** path may still be missed — no implied exhaustiveness, and the residual gap stays visible in the report rather than being disguised as a clean "no match found". If nothing matches, say so plainly and show the nearest misses with their exact gaps — a near-miss with a 2dB gap is actionable information (the user may relax the spec).

### Step 5 — Final audit before sending

Run this checklist; fix anything that fails before reporting:

- [ ] Every ✅/⚠️ part: every required parameter has an actual value, min/typ/max label, margin, and a working datasheet/catalog link. (Exception: a ⚠️ access-blocked match carries site values, its guard-band-clear margins, and an aggregator link in place of the unreachable datasheet, plus the "not verified" flag.)
- [ ] Every ✅/⚠️ part survived Step 3.5 re-verification, and the mode used (subagent / single-agent) is stated in the coverage statement. (A ⚠️ access-blocked match is re-verified against site values, not a datasheet, and stays ⚠️.)
- [ ] No part reached a datasheet ❌ on snippet evidence alone — every datasheet ❌ rests on an actual datasheet value. (Step 2.7 site-screen rejects and access-blocked *unverifiable* parts are the only non-datasheet entries in the rejected sheet, each labelled as such.)
- [ ] Any candidate dropped at the Step 2.7 site screen is logged in the coverage sheet as *rejected at site screen*, with its failing site parameter and value — not flagged "not datasheet-verified". The "not datasheet-verified" flag is used only for a part that passed **every** specified site-checkable parameter at 100% but whose datasheet was not checked.
- [ ] A part whose datasheet was inaccessible: alternative sources were tried and logged, and it was classified by the site-checkable / datasheet-only rule — ⚠️ *not-verified match* in התאמות if every specified parameter is site-checkable and clears the spec beyond its guard band; otherwise an *unverifiable reject* in נבדקו ונפסלו marked "consider contacting the manufacturer". Never auto-dumped into rejected, never shown as ✅.
- [ ] Every ❌ part has the specific failing parameter and its actual value — except an access-blocked *unverifiable* part, which instead names the datasheet-only parameter that could not be checked and flags "consider contacting the manufacturer".
- [ ] No part is from an excluded vendor (including legacy brands now hosted on excluded domains).
- [ ] All three discovery paths ran for this category, and the coverage statement reports each one's outcome (Path A parametric — pages paged; Path B traversal — waves + new vendors found; Path C cache sweep). A path that was skipped is named as such, not silently omitted.
- [ ] The coverage statement states plainly that coverage is bounded by these sources and a vendor absent from every path may still be missed — the residual gap is visible, not hidden behind a clean "no match".
- [ ] Any vendor newly discovered by Path A or Path B was appended to the vendor cache with its access metadata.
- [ ] The clarifications from Step 1 are reflected (e.g. if a bound is hard, no match exceeds it).
- [ ] The loaded component module's sanity checks were applied and any violation was re-checked against the source.
- [ ] Numbers in the chat table match the Excel exactly.
- [ ] The Excel has all three sheets (התאמות / נבדקו ונפסלו / יומן כיסוי) — even if a sheet is empty apart from its header.
- [ ] The יומן כיסוי manufacturers table has the **"שאילתה שנשלחה"** column, and every vendor/source that was actually checked shows the real query string sent to it (keyword string, Path A filter set, or catalog search term/URL) — not a placeholder; a *not covered* source may leave it empty.
- [ ] The יומן כיסוי manufacturers table is **exhaustive**: every vendor touched at least once through any path has its own row in the identical format, including vendors that returned nothing (`checked/no candidates`) — none omitted for being empty.
- [ ] Every parametric aggregator (Path A) is **expanded**: each vendor its filtered results surfaced has its own manufacturer row, not hidden inside a single "everything.rf — N parts" line.
- [ ] If 0–1 matches: an extra Path B (part-graph) wave was run and the coverage statement says so.

## Efficiency notes

- Phase searches in two stages: cheap wide search → **Step 2.7 site screen** (checks only the query's site-checkable parameters, per the component module) → datasheet fetches for survivors only. This is the main token saver: datasheets are opened only for parts that pass the site screen. Every Stage-1 reject is logged as *rejected at site screen* (its failing site parameter and value) — not "not datasheet-verified"; datasheet-only, unexposed, borderline, or absent parameters always promote, never drop.
- If the user names a vendor missing from the cache below, add it to the cache (Path C) for the rest of the session with whatever access metadata you learn — the same auto-append rule Paths A/B use.

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

### VENDOR CACHE — access knowledge for known manufacturers (formerly the sweep list)

This is a **cache**, not the source of truth for which vendors exist. The universe of vendors is defined by Path A (parametric aggregators) and Path B (part-graph traversal); this list's only job is to make access to an *already-known* vendor cheap and reliable — per-vendor access metadata and institutional knowledge (domain, catalog URL, parse/access notes, category hints). Path C sweeps it. Generic web search misses many of these (poor indexing), so checking their catalogs directly still matters — but a vendor being absent here no longer means it won't be found.

The cache is **self-growing** — no human maintains it:

- Whenever Path A or Path B discovers a vendor NOT already listed here, append it with whatever access metadata was learned (domain, catalog URL, parse/access notes, component categories). The cache grows automatically toward completeness for the categories actually searched.
- The cache MAY be seeded or refreshed from **everything.rf's Companies directory** for the relevant category — it maintains a live per-category list of manufacturers.
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