---
name: hms-ha-intelligence-stage1
description: "Stage 1 of HMS housing association intelligence. Captures organisation-level signals for a UK housing association: HMS supplier and product, vendor M&A exposure, contract lifecycle stage, regulatory picture (RSH grades, Ombudsman), financial picture, operational signals (Awaab readiness, TSM, repairs), strategic direction, procurement signals, competitor presence, lateral self-disclosure, and people identification for Stage 2. Outputs a structured research notes file. First of three stages; Stage 2 validates people, Stage 3 composes opportunity signals and publishes to Pipedriven. Use when profiling any UK HA at the org-research stage. Trigger on HA HMS intelligence Stage 1, organisation research notes, HMS performance signal capture, registered provider HMS research, vendor M&A in social housing (Civica/Blackstone, MRI/Capita, Aareon), HA contract lifecycle, HA regulatory and financial signals."
---

# HA Skill, Stage 1: Organisation Research

**Purpose.** Build the org-level intelligence picture for a UK housing association. Capture tag values for five of the seven Pipedriven sections (`overview`, `strategicDirection`, `corporateRisks`, `challenges`, `financialHealth`), plus procurement signals, competitor presence, lateral self-disclosure, people identification, raw gaps log, and primary trigger candidate. Output a research notes file for Stage 2 and Stage 3.

---

## Operating mode

Single conversation per org. Web search active. Pipedriven MCP active for Step A1.

**Execute inline. DO NOT dispatch to `launch_extended_search_task`, `launch_research_agent`, or any research delegate.** Deep research disabled at UI level.

**Output saved to** `/mnt/user-data/outputs/{ORG_NAME}_Stage1_Research_Notes.md` or the equivalent Cowork local folder.

---

## Core concepts

- **Search budget.** MAX 3 query variants per tag. After 3, declare `Unknown` with reason and route to raw gaps log. Once resolved, don't re-search.
- **Stage 1 ceilings.** 60 web searches total. 12 web_fetch total (5 of which may be in B6).
- **Defensive count.** Before each search, count searches run total and on the current tag. Stop at 60 total or 3 per tag.
- **Source hierarchy.** Tier 1 (official: org site, RSH register, Companies House, Find a Tender, regulator publications). Tier 2 (supplier-published, sector body publications). Tier 3 (trade press: Inside Housing, Social Housing). Tier 4 (general press, blogs). Named officer quotes inside Tier 4 count as Tier 2 embedded.
- **Value discipline (schema v0.4).** Closed-list values exact. Living-list values by established name. Structured free-text by format.
- **Unknown handling.** Every in-scope tag has an explicit value: researched, `Unknown` with reason, `Not applicable`, or `None`. Never blank.
- **CRM as relationship history only.** Pipedriven is consulted in A1 for warmness, account owner, prior engagement. NEVER a source of current roles. CRM contact names flagged for Stage 2 validation.
- **Broader lens.** When capturing system-mediated signals, note operating context (process, operating model, capability, decision-making) in evidence notes where the evidence supports it. The aim is a rounded picture for the full 4OC offer, not HMS-narrow.
- **Raw gaps log.** Every item that hit budget without resolution: what couldn't be confirmed, sources tried, notes. Stage 3 applies four-bin discipline.

---

## Phase A: Foundation

### Step A1. Master Target List and CRM cross-reference

_Tags produced._ CRM warmness, Account owner.

_Sources._ HMS Master Target List entry; Pipedriven `search_organizations` and `get_organization_overview` for warmness, owner, prior engagement.

_Discipline._ Pipedriven contact records are not a source of current roles. Any CRM contact name is flagged for Stage 2 validation.

### Step A2. Organisation snapshot and group structure

_Tags produced._ Organisation type, Region, Stock size. Group structure as narrative note.

_Sources._ Annual report, RSH register, Companies House, "About us" page.

_Search queries._

- `{Org name} group structure`
- `{Org name} subsidiary OR member entity`
- `{Org name} annual report 2024 OR 2025`
  _Discipline._
- Single entity, has parent (named), has subsidiaries (list with type), or recently merged (capture for B10).

### Step A3. HMS supplier and product identification

_Tags produced._ HMS Supplier, HMS Product, HMS Platform Type, SI Partner.

_Sources._ Supplier case studies (Tier 2), Find a Tender (Tier 1), digital strategy or annual report (Tier 1), trade press (Tier 3 corroboration).

_Search queries._

- `{Org name} {HMS Product candidate}` (e.g., "Riverside NEC Housing")
- `{Org name} housing management system`
- `site:findatender.service.gov.uk {Org name} housing`
  _Discipline._
- HMS Supplier = corporate parent. HMS Product = specific platform. Separate tags.
- D365/Salesforce/SAP → also capture SI Partner.
- Multiple systems → primary HMS + "Mixed", note others in narrative.

### Step A4. Vendor M&A exposure

_Tags produced._ Vendor M&A Affected (multi-valued, in `corporateRisks`).

_Known events._ Civica/Blackstone 2024, MRI/Capita 2025, Aareon/MIS Active 2022, Aareon/HomeMaster 2025.

_Search budget._ Zero for the events themselves. 1-2 per event to confirm THIS org is on the affected platform.

_Discipline._ Multi-valued; an org can be exposed to more than one.

### Step A5. HMS contract details

_Tags produced._ HMS Contract start date, end date, status, route, Contract lifecycle stage.

_Sources._ Find a Tender (post-Feb 2025), CDP/Contracts Finder (pre-Feb 2025), framework portals (CCS, ESPO, YPO), annual procurement report.

_Search queries._

- `{Org name} {HMS Product} contract notice OR award`
- `{Org name} housing management system tender`
- `site:findatender.service.gov.uk {Org name} housing`
  _Discipline._
- Start date found → derive lifecycle stage from schema v0.4 closed list.
- End date not found, supplier known → raw gap; lifecycle = "unknown".
- Pre-PA2023 with no public trace → declare "Pre-PA2023 invisible", don't waste budget.

---

## Phase B: Signal capture

### Step B6. Lateral self-disclosure scan

_Purpose._ Surface org-published documents that inform downstream signal capture. Self-disclosed evidence is stronger than third-party.

_Documents to find._ TSM return, Awaab's Law readiness, damp/mould action plan, complaints annual report, customer experience review, digital strategy, ASB strategy, Consumer Standards self-assessment, Tenant Involvement self-assessment.

_Search budget._ 1-2 queries per document type, capped at 12 searches total. Document either exists or doesn't; if not found in 2 queries, move on.

_Fetch budget._ MAX 5 web_fetch across the whole B6 scan. Priority if >5 surfaced: TSM, Awaab readiness, complaints annual report, customer experience strategy, damp/mould plan.

_Discipline._ Documents found are Tier 1 sources. Cite by name and date inline downstream.

### Step B7. Regulatory picture (corporateRisks tags)

_Tags produced._ RSH Consumer grade, RSH Governance grade, RSH Viability grade, RSH Regulatory notice, Housing Ombudsman maladministration count, severe count, dominant categories, External auditor, Audit opinion. (Government intervention, PIR: LG only, mark "Not applicable" for HA.)

_Sources._ RSH register (Tier 1), Housing Ombudsman published decisions (Tier 1), annual report for auditor/opinion, trade press (corroboration).

_Search queries._

- `{Org name} RSH regulatory judgement` or browse RSH register directly
- `{Org name} Housing Ombudsman determination`
- `{Org name} severe maladministration`
  _Discipline._
- RSH grades use exact closed-list (C1-C4 / G1-G4 / V1-V3).
- Ombudsman counts: 24-month rolling. If unable to count precisely, declare Unknown with reason.
- Recent grade change in 12 months: capture explicitly with date.

### Step B8. Financial picture (financialHealth tags)

_Tags produced (HA)._ Bond issuer status, Credit rating, Capital programme position. (Section 114, EFS, MTFS, HRA: LG only, mark "Not applicable" for HA.)

_Sources._ Annual financial statements (Tier 1), rating agency reports (Tier 1), bond prospectuses (Tier 1), Inside Housing/Social Housing (corroboration).

_Search queries._

- `{Org name} credit rating Moody's OR S&P OR Fitch`
- `{Org name} bond issuance`
- `{Org name} annual report financial statements`
  _Discipline._
- Bond issuer: most recent issuance year and value. "Not an issuer" if confirmed.
- Credit rating multi-valued: one per agency, format `{Agency} {Grade} {Outlook} ({Date})`.

### Step B9. Operational signals (challenges tags)

_Tags produced._ System landscape, Awaab's Law readiness, TSM overall satisfaction, TSM trend, Post-merger integration status, Repairs performance, Damp and mould response capability.

_Sources._ B6 self-disclosure documents, annual report, RSH aggregated TSM data, press and Ombudsman cases for damp/mould.

_Search queries (where not covered in B6)._

- `{Org name} TSM 2024 OR 2025`
- `{Org name} repairs performance`
- `{Org name} damp mould complaint`
  _Discipline._
- TSM overall satisfaction: latest published year.
- TSM trend: 3-year window. One year only → "Not enough data".
- Awaab's Law readiness: Phase 1 / Phase 2 status. Source ideally B6 document.
- Post-merger integration: only if merger flagged. "Not applicable" otherwise.
- Broader lens: note whether signal is system-mediated, process-mediated, capability-mediated, or operating-model-mediated.

### Step B10. Strategic direction (strategicDirection tags)

_Tags produced (HA)._ Recent merger, Recent insourcing, Named transformation programme, Recent governance change. (LGR status: LG only, "Not applicable" for HA.)

_Sources._ Annual report, Inside Housing/Social Housing, board papers, LinkedIn for executive moves (identifying only).

_Search queries._

- `{Org name} merger OR partnership 2024 OR 2025`
- `{Org name} new CEO OR new chair OR appointed`
- `{Org name} transformation programme`
  _Discipline._
- Recent governance change: last 18 months only. Multi-valued. Format "Role; date; name; prior employer if known".
- Named transformation programme: named programmes only. Generic references → "None".
- Recent merger / insourcing: date and parties.

### Step B11. Procurement signals

_Purpose._ Procurement signals beyond the HMS contract: other tech procurements, frameworks signed, tenders in flight.

_Tags produced._ None directly. Raw notes feed Stage 3.

_Sources._ Find a Tender, CDP/Contracts Finder, org procurement page.

_Search queries._

- `site:findatender.service.gov.uk {Org name}`
- `{Org name} tender OR procurement notice`
- `{Org name} framework call-off`
  _Discipline._
- Capture notices, awards, PINs with dates, values, routes.
- Flag adjacent tech procurements (asset, CRM, repairs) as system landscape signals.
- Note framework usage patterns (CCS, ESPO, direct).

### Step B12. Competitor presence

_Tags produced._ Competitor presence draft (multi-valued; Stage 3 finalises into `opportunitySignals`).

_Sources._

- HMS Competitor Research Log (PRIMARY) in project folder (`HMS_Competitor_Research_Log.md`)
- Supplementary web search only if log has no entries for this org
- HMS Competitor Universe for tier classification
  _Search queries (supplementary only)._
- `{Org name} consulting OR advisory 2024 OR 2025`
- `{Org name} {known competitor name}` (when targeted)
  _Search budget._ Zero if log has entries. 2-3 variants if log has none.

_Discipline._

- Log is source of truth. Don't re-research what's there.
- Tier from Competitor Universe (Housing tier for HA).
- Format: `{Firm} ({Housing tier}); {Nature}`.

---

## Phase C: People and compile

### Step C13. People identification

_Purpose._ Identify named officers for Stage 2 validation. Operational layer is the PRIMARY target for BD; exec team is secondary.

**Operational Leads** (PRIMARY focus; multiple per org, service-specific)

- Assistant Director or Head of Resident Services (tenancy, housing management)
- Assistant Director or Head of Property Services / Maintenance (repairs, asset, planned works)
- Assistant Director or Head of Customer Services / Customer Experience
- Assistant Director or Head of Income / Rents / Allocations
- Capture each where the org structure has them as distinct functions.
  **Technical Lead / System Owner** (one or two per org)
- Head of Housing Systems, Systems Manager, Housing IT Manager (the day-to-day HMS owner)
- CIO / IT Director (at HA level, CIO is close to operational HMS decisions; capture as Technical Lead, not oversight)
  **Strategic Lead** (one per org; capture LAST as the executive umbrella, not the starting point)
- Director of Housing, Executive Director Housing, Director of Resident Services. Top of the housing function.
  Titles vary across HA. Persona is defined by function, not by exact title.

_Sources (priority order)._ (1) LinkedIn (`site:linkedin.com` searches for AD/Head titles). (2) RocketReach, TheOrg.com, LeadIQ aggregators. (3) Sector press for appointments (Inside Housing, Social Housing, Housing Today). (4) Org website Executive/Leadership pages and annual report (LAST RESORT for operational layer). (5) Companies House for directors only.

_Search queries (web_search, NOT web_fetch)._

- `site:linkedin.com {Org name} "Head of" OR "Assistant Director" Housing OR "Resident Services"`
- `site:linkedin.com {Org name} "Head of" OR "Director of" Property OR Repairs OR Maintenance OR Asset`
- `site:linkedin.com {Org name} "Head of" OR "Director of" Customer Services OR "Customer Experience"`
- `site:linkedin.com {Org name} "Head of" Income OR Rents OR Allocations`
- `site:linkedin.com {Org name} "Head of" "Housing Systems" OR "Housing IT" OR "Head of IT"`
- `{Org name} site:theorg.com` OR `{Org name} site:rocketreach.co` (aggregator backup)
  _Search budget._ 3 query variants per persona gap. After 3, route to raw gaps log and move on.

_Captured per person._ Name, indicative current role, source (with date), persona category, "CRM-sourced; pending validation" flag if from A1.

_Discipline._

- Identification only. Names found here are HYPOTHESES for Stage 2.
- Search order: Operational Leads (AD/Head layer) FIRST via LinkedIn/aggregator searches, then Technical Lead, then Strategic Lead/C-suite LAST.
- Use web_search on LinkedIn and aggregators FIRST. Do NOT web_fetch the org's executive page until LinkedIn/aggregator searches are exhausted.
- Finding 5 C-suite names does NOT complete this step. Operational layer is the priority.

### Step C14. Compile output file

_Output._ `{ORG_NAME}_Stage1_Research_Notes.md` saved to `/mnt/user-data/outputs/`.

_Structure._

1. Header (org name, date, researcher or Cowork task ID)
2. Tag block + evidence notes for each of: `overview`, `strategicDirection`, `corporateRisks`, `challenges`, `financialHealth`
3. Procurement signals (raw; Stage 3 composes)
4. Competitor presence (draft multi-value; Stage 3 finalises)
5. People Identified table (name, role, source, persona, validation flag)
6. Raw gaps log (what couldn't be confirmed; sources tried; notes)
7. Sources (Tier 1-4, with URLs and dates)
8. Primary trigger candidate: one or two compound triggers with one-line evidence pointer each; "None material; monitor only" if none surfaces.
   _Discipline._ No web search at this step. Output assembly only. Nothing filled by guesswork.

---

## Stage 1 output file template

```markdown
# {ORG_NAME}, Stage 1 Research Notes

**Date.** {YYYY-MM-DD}
**Researcher.** {Name or Cowork task ID}
**Schema.** v0.4

---

## overview

[All overview tags + Evidence notes]

## strategicDirection

[All strategicDirection tags + Evidence notes]

## corporateRisks

[All corporateRisks tags + Evidence notes]

## challenges

[All challenges tags + Evidence notes (with broader lens context where relevant)]

## financialHealth

[All financialHealth tags + Evidence notes]

## Procurement signals (raw; for Stage 3)

- ...

## Competitor presence (draft for Stage 3)

- {Firm} ({Housing tier}); {Nature}

## People Identified (for Stage 2 validation)

| Name | Indicative role | Source | Persona | Validation flag |
| ---- | --------------- | ------ | ------- | --------------- |

## Raw gaps log

| What couldn't be confirmed | Sources tried | Notes |
| -------------------------- | ------------- | ----- |

## Sources

- [Tier 1] ... ({URL}, {Date})

## Primary trigger candidate (for Stage 3)

- **Trigger 1**: [name from trigger taxonomy]. Evidence: [one-line pointer].
- **Trigger 2** (if compound): [name]. Evidence: [one-line pointer].
```

---

## Quality checks before handoff

1. All 36 in-scope HA tags have explicit values.
2. Tag values conform to schema v0.4.
3. Evidence notes cite sources inline.
4. Where system-mediated signals captured, broader lens context recorded.
5. People Identified covers Operational Leads (PRIMARY), Technical Lead, Strategic Lead (LAST), where the org has them.
6. Raw gaps log is non-empty if anything unresolved (or explicitly "no unresolved items").
7. Sources list comprehensive.
8. Primary trigger candidate stated.
   If any check fails, complete or log the gap before producing the final output.
