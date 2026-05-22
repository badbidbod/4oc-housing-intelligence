---
name: hms-ha-intelligence-stage2
description: "Stage 2 of HMS housing association intelligence. Takes a Stage 1 Research Notes file and validates the people identified, producing a Validated Contacts file ready for Stage 3 publishing. Validates LinkedIn status, current title and scope, persona assignment. Discovers the org's email pattern and applies it per person with explicit confidence tags (Verified / Inferred high / Inferred medium / Inferred best-guess). Captures recent LinkedIn activity for outreach talking points. Flags departures separately. Every person populated with an email; no blanks. Use after Stage 1 has produced a Research Notes file for the org. Trigger on HA HMS intelligence Stage 2, validate HA contacts, populate emails for HA contacts, HA people validation, HA outreach contact list preparation."
---

# HA Skill, Stage 2: People Validation

**Skill.** `hms-ha-intelligence-stage2`, Stage 2 of 3.

**Status.** Draft, 21 May 2026. Companion to Tag Schema v0.4, HA Three-Stage Architecture v0.3, and `hms-ha-intelligence-stage1` (Stage 1).

**Purpose.** Take the People Identified table from a Stage 1 Research Notes file and validate each person to the point where they're ready for outreach. Produce a Validated Contacts file consumable by Stage 3 (profile assembly + Pipedriven publish via `save_org_people` and `publish_org_intelligence`).

**Scope, out.** Org-level signal capture (Stage 1). opportunitySignals composition (Stage 3). Pipedriven publishing (Stage 3). Researching new persona gaps not surfaced in Stage 1 (Stage 1's job).

---

## Operating mode

Single conversation per org. Web search active. Pipedriven MCP active for CRM cross-reference. Output saved to `/mnt/user-data/outputs/{ORG_NAME}_Stage2_Validated_Contacts.md`.

**Execute inline in this conversation. DO NOT dispatch to `launch_extended_search_task` or any research subagent.** The search budget discipline only holds when one conversation does the searches itself. Deep research should be disabled at the user interface level when this skill runs.

---

## Inputs

1. **Stage 1 Research Notes file** for the org. Standard locations to check, in order:
   - `/mnt/user-data/uploads/{ORG_NAME}_Stage1_Research_Notes.md`
   - `/mnt/project/{ORG_NAME}_Stage1_Research_Notes.md`
   - `/mnt/user-data/outputs/{ORG_NAME}_Stage1_Research_Notes.md`
   - Or the path explicitly provided by the user.
2. **Pipedriven contacts** for the org (via `search_organizations` → `list_organization_contacts`). Used for CRM cross-reference, not as a source of current state.

---

## Outputs

`{ORG_NAME}_Stage2_Validated_Contacts.md` saved to `/mnt/user-data/outputs/`. One block per validated person, plus org-level email pattern, departures section, and summary stats.

---

## Core concepts

**Search budget.** MAXIMUM four search variants per person across all checks (LinkedIn status + role + email + recent activity). After four searches without resolution, accept what's found, flag confidence accordingly, and move on. Once a fact is captured (title, employer, email), DO NOT re-search.

**Stage 2 total search ceiling.** No more than 50 web searches across the whole stage. No more than 6 web_fetch operations. Most evidence comes from search result snippets; fetch only when essential.

**Defensive check before every search.** Mentally count total searches so far and searches against the current person. Hard stops: 50 total, 4 per person.

**Email discipline.** Every person populated with an email. Confidence tagged. Four tiers:

- **Verified**: found publicly listed for this specific person (LinkedIn bio, press release, conference contact, signed public statement).
- **Inferred (high)**: org pattern verified by 2+ published examples; applied here.
- **Inferred (medium)**: org pattern based on 1 published example; applied here.
- **Inferred (best-guess)**: no published examples found; HA sector default `firstname.lastname@{org-domain}` applied; flagged as a guess.
  Never leave an email blank. Outreach can risk-manage on the confidence tag; a bounce becomes the signal for a manual lookup on that individual.

**LinkedIn as primary source.** For validating current title, scope, and recent activity, LinkedIn is the most reliable single source. Use `site:linkedin.com {person name} {org name}` and variants. If a person can't be found on LinkedIn under the current org name, try the legacy brand (post-merger).

**Persona reconciliation.** The Stage 1 persona assignment is a hypothesis. Stage 2 confirms or adjusts based on validated scope. A "Strategic Lead" hypothesis may downgrade to "Operational Lead" if the validated scope is service-specific rather than function-wide, or vice versa.

**Departures captured separately.** If a person is no longer at the org, do NOT include them in the active contacts list. Capture in a "Departures and onward moves" section with where they went. People who moved to other HAs may be useful contacts at their new org and worth flagging for the BD team.

**CRM contacts validated externally.** CRM contacts from Stage 1 (flagged "CRM-sourced; pending validation") must be confirmed via external sources (LinkedIn / org website) before being treated as current.

**Broader lens.** When validating roles, capture context that informs outreach: recent role change, scope expansion, conference appearance, public statement on a relevant topic, named in a regulatory document. These become talking points for Stage 3.

**Raw gaps log.** Tracks people from Stage 1 who couldn't be validated, and persona gaps from Stage 1 that Stage 2 was asked to fill but couldn't. Four columns: who, sources tried, status, notes.

---

## Steps

### Step S1. Load Stage 1 output

_Purpose._ Read the Stage 1 Research Notes file. Extract people, persona gaps, CRM contacts, and org context.

_Process._

1. Locate the Stage 1 file in the standard locations or the user-provided path.
2. Read the file in full.
3. Extract:
   - **People Identified table** (name, indicative role, source, persona, validation flag).
   - **Persona gaps** from the Stage 1 raw gaps log.
   - **CRM-sourced contacts** (rows tagged with "CRM-sourced; pending validation").
   - **Org context**: name, domain, post-merger status (and legacy brand names if applicable), stock size (informs whether to push hard on the operational layer), HMS supplier (informs Technical Lead validation).
     _Discipline._

- If the file isn't found, stop and ask the user for the path. Don't proceed without it.
- If the People Identified table is empty or absent, Stage 1 didn't complete properly. Flag and stop.

### Step S2. Org-level email pattern discovery and verification

_Purpose._ Determine the org's primary email pattern. Verify with as many published examples as possible (target: 2+). Confidence informs per-person email tagging in S5.

_Sources (priority order)._

1. Org's contact pages and press releases (`site:{org-domain} @{org-domain}`).
2. Procurement notices on Find a Tender (`site:find-tender.service.gov.uk {Org name}` returns notices that often include the buyer contact email).
3. RocketReach pattern page for the org (`site:rocketreach.co {Org name}`).
4. Hunter.io domain page (`site:hunter.io {org-domain}`).
5. Investor relations pages and bond prospectuses (CFO / investor contact).
6. Annual report key personnel section.
   _Search queries._

- `site:{org-domain} @{org-domain}`. Finds emails published on the org's own site
- `"{org-domain}" email contact`. Finds emails referenced elsewhere
- `site:rocketreach.co "{Org name}"`. RocketReach pattern page
- `site:hunter.io {org-domain}`. Hunter.io domain summary
- `"{Org name}" procurement contact email`. Buyer contact on tender notices
  _Search budget._ Up to 5 searches for org-level pattern discovery.

_Multi-domain orgs (post-merger)._ If the org has multiple legacy brands, repeat for each domain (e.g., `@bromford.co.uk`, `@flagship-housing.co.uk`, `@livewest.co.uk`, plus any new merged-entity domain). Capture which domain applies to which legacy population. People still under their legacy brand likely retain the legacy domain until full migration.

_Captured._

- **Primary pattern**: e.g., `firstname.lastname@{org-domain}`
- **Number of verified examples**: 0 / 1 / 2+
- **Verified examples** (with URLs)
- **Pattern confidence**: Verified (2+ examples) / Limited (1 example) / Unknown (0 examples; sector default to be used)
- **Domain notes**: which domains apply to which legacy population

### Step S3. Per-person LinkedIn validation

_Purpose._ For each named person from Stage 1 (and CRM-sourced contacts), confirm still in post, current title and scope, reporting line where visible.

_Per person:_

1. Search `site:linkedin.com "{person name}" {org name}`.
2. Confirm the LinkedIn profile shows the org as current employer.
3. Capture: current title (as displayed on LinkedIn), scope summary (from headline and recent role description), tenure (start date in role), reporting line (where visible from recent activity).
4. Flag any of: departure, role change, ineligibility (e.g., long-term leave indicated).
5. If LinkedIn search returns nothing under the current org name, try the legacy brand. People stay under their legacy brand until they update profiles.
   _Search budget._ Up to 2 searches per person for LinkedIn validation.

_Discipline gates._

- LinkedIn profile must explicitly show the org as current employer (not "former" or "past"). A profile that says "previously [the org]" means the person has left.
- "About 6 months ago" tenure on a role recently advertised by the new C-suite is consistent with being a recent appointment to the operational layer, not a departure.
- CRM contacts flagged "pending validation" from Stage 1 get the same LinkedIn check as named-from-research contacts.

### Step S4. Persona reconciliation

_Purpose._ Confirm or adjust the Stage 1 persona hypothesis based on validated scope.

_Personas (carried from Stage 1, function-based):_

- **Strategic Lead**: heads the housing function as a whole.
- **Operational Lead**: owns delivery of a specific function (Resident, Property, Customer, Income, etc.).
- **Technical Lead**: owns the HMS day-to-day.
- **Board / Executive**: governance role.
- **Other**: relevant but doesn't fit above (e.g., Strategic Advisor, recent appointee with broad portfolio).
  _Decision logic._
- If validated scope is function-wide: Strategic Lead.
- If validated scope is service-specific (one function only): Operational Lead, named by function (e.g., "Operational Lead, Property Services").
- If validated scope is HMS/systems-focused: Technical Lead.
- If the person is a Board member (NED, Chair, audit committee chair): Board.
- Adjust the Stage 1 hypothesis where the validated scope contradicts it. Record both the Stage 1 hypothesis and the Stage 2 confirmed persona.

### Step S5. Per-person email assignment

_Purpose._ Populate an email for every validated person, with confidence tag.

_Per person:_

1. First, search for a publicly listed email for THIS person (`"{person name}" {org-domain}`, `"{person name}" email`, `site:linkedin.com {person name} contact info`). If found, status = **Verified**.
2. If no published email found, apply the org's primary pattern from S2:
   - If S2 confidence = Verified (2+ examples): status = **Inferred (high)**.
   - If S2 confidence = Limited (1 example): status = **Inferred (medium)**.
   - If S2 confidence = Unknown (0 examples): apply HA sector default `firstname.lastname@{org-domain}`, status = **Inferred (best-guess)**.
3. For multi-domain orgs, apply the domain matching the person's legacy brand.
4. Handle special cases:
   - **Hyphenated surnames**: try with hyphen first; some orgs strip it.
   - **Compound first names**: try most common form.
   - **Senior people with PA / executive assistant**: capture both addresses if the PA address is the route in.
     _Search budget._ Up to 1 search per person for verified email lookup. If not found, infer.

_Captured per person:_

- Email address
- Status (Verified / Inferred high / Inferred medium / Inferred best-guess)
- Verification source (URL, date) if Verified
- Pattern applied if Inferred
- Domain reasoning if multi-domain org

### Step S6. Recent activity capture (talking points)

_Purpose._ Pull recent LinkedIn activity per person. Used by Stage 3 to compose outreach talking points.

_Per person:_

1. Search `site:linkedin.com "{person name}" {org name}` and review the recent activity / posts visible in search snippets.
2. Capture 2-3 themes from the last 3-6 months. Examples: "Posted on Awaab's Law readiness", "Spoke at Housing 2026 on damp and mould transformation", "Promoted hire for Director of Repairs".
3. Note any conference appearances or panel participation.
4. Note any public statement on a topic relevant to 4OC's services (system change, regulatory recovery, transformation, procurement, data migration).
   _Search budget._ Up to 1 search per person for recent activity (often this comes from the same search as S3 LinkedIn validation; combine where possible).

_Discipline._

- Snippets only; no need to web_fetch full LinkedIn pages unless a specific post is highly relevant and the snippet is cut off.
- Keep themes tight: 2-3 max. More than that becomes noise.

### Step S7. New people surfaced during validation

_Purpose._ During LinkedIn validation (S3) and recent activity capture (S6), names appear who weren't in Stage 1 (reporting lines, mentioned colleagues, recent appointment announcements). Capture them.

_Per new person surfaced:_

1. Apply the same S3 validation discipline (LinkedIn confirmation).
2. Apply S4 persona reconciliation.
3. Apply S5 email assignment.
4. Apply S6 recent activity capture.
   _Search budget._ Each new person uses the same per-person budget (4 search variants total across all their checks).

_Discipline._

- Cap new people surfaced at 5. Beyond that, route to a "further surfaced names" list without full validation (Stage 3 or a follow-up Stage 2 run can address).
- New people often fill persona gaps from Stage 1's raw gaps log. Match them to gaps where applicable.

### Step S8. Compile output

_Purpose._ Assemble all validated contacts into the Stage 2 output file.

_Output file structure._ See output template below.

_Final discipline gates._

- Validation summary block populated at the top of the output (counts of validated, departed, newly surfaced; persona coverage; email confidence breakdown).
- Every active contact has an email with a status tag. Verify by counting: number of active contacts == number of populated emails.
- Departed people in separate section, not in active contacts list.
- Persona gaps from Stage 1 either filled (by new people surfaced in S7) or carried forward to the Stage 2 raw gaps log.
- Sources cited per fact.

---

## Output file template

```markdown
# {ORG_NAME}, Stage 2 Validated Contacts

**Date.** {YYYY-MM-DD}
**Researcher.** Claude (inline, hms-ha-intelligence-stage2 skill)
**Input.** {ORG_NAME}\_Stage1_Research_Notes.md (Stage 1 reference)

---

## Validation summary

- **Stage 1 people validated:** {N of M Stage 1 people confirmed as active and approachable}
- **Stage 1 people departed:** {N moved to departures with onward moves noted}
- **New people surfaced during validation:** {N found via LinkedIn or pattern discovery, not in Stage 1}
- **Persona coverage:** Strategic Lead {Yes/No}, Operational Lead(s) {N functions covered}, Technical Lead {Yes/No}, Board {Yes/No/Not approached}
- **Email coverage:** {N} active contacts populated with email (no blanks); confidence breakdown: Verified {N}, Inferred high {N}, Inferred medium {N}, Inferred best-guess {N}

---

## Org-level email pattern

**Primary pattern:** `firstname.lastname@{org-domain}`
**Pattern confidence:** Verified (2+ examples) / Limited (1 example) / Unknown
**Verified examples:**

1. {published email} (source: {URL}, date: {date})
2. {published email} (source: {URL}, date: {date})

**Multi-domain notes (if applicable):**

- {legacy brand 1}: `@{legacy-domain-1}` applies to ex-{legacy brand 1} staff
- {legacy brand 2}: `@{legacy-domain-2}` applies to ex-{legacy brand 2} staff
- {merged entity}: `@{new-domain}` applies to new appointments post-merger

---

## Validated contacts (active)

### {Name}

- **Validated title:** {current title from LinkedIn / public source}
- **Scope:** {brief summary, e.g., "Housing management, lettings, estates, ~60,000 properties"}
- **Persona:** {Strategic Lead / Operational Lead, {function} / Technical Lead / Board / Other}
- **Stage 1 persona hypothesis:** {what Stage 1 said, if different}
- **Email:** {address}
- **Email status:** {Verified / Inferred high / Inferred medium / Inferred best-guess}
- **Email source (if Verified):** {URL, date}
- **Domain reasoning (multi-domain):** {which legacy brand domain applies and why}
- **LinkedIn:** {URL}
- **Tenure in role:** {since {date}}
- **Reporting line (if visible):** {to {name and role}}
- **Recent activity themes (last 3-6 months):**
  - {Theme 1}
  - {Theme 2}
  - {Theme 3}
- **Conference / public appearances:** {any}
- **Relevant public statements:** {any directly relevant to 4OC's services}
- **Notes:** {anything that affects outreach}

(Repeat for each active contact.)

---

## Departures and onward moves (do not approach at this org)

### {Name}

- **Former role:** {title}
- **Departed:** {date}
- **Where they went:** {new org and role, if known}
- **Onward contact value:** {Yes/No, with brief note; e.g., "Now CEO at ISHA, potential 4OC contact at ISHA"}

(Repeat for each departure.)

---

## Persona coverage summary

| Persona                             | Filled | Names   |
| ----------------------------------- | ------ | ------- |
| Strategic Lead                      | Yes/No | {names} |
| Operational Lead, Resident Services | Yes/No | {names} |
| Operational Lead, Property Services | Yes/No | {names} |
| Operational Lead, Customer Services | Yes/No | {names} |
| Operational Lead, Income            | Yes/No | {names} |
| Technical Lead                      | Yes/No | {names} |
| Board                               | Yes/No | {names} |

---

## Raw gaps log

| What couldn't be confirmed           | Sources tried                              | Status     | Notes                                                                      |
| ------------------------------------ | ------------------------------------------ | ---------- | -------------------------------------------------------------------------- |
| {e.g., Current CIO post-Rajiv Peter} | {LinkedIn-by-company; NHG leadership page} | Unresolved | {LinkedIn shows page still names Rajiv; org may not have replaced him yet} |

---

## Summary stats

- Active contacts validated: {count}
- Emails populated: {count} (must equal active contacts)
- Verified emails: {count}
- Inferred (high): {count}
- Inferred (medium): {count}
- Inferred (best-guess): {count}
- Departures captured: {count}
- New people surfaced during validation: {count}
- Persona gaps filled: {count of {total Stage 1 gaps}}

---

## Sources

- {Tier 1/2/3/4 sources used, with URLs and dates}
```

---

## Quality checks before output

1. Every active contact has an email. Run the count: active contacts == populated emails. If less, populate the missing ones (apply best-guess if necessary) before saving the file.
2. Every email has a status tag. No bare emails.
3. Departures are in the Departures section, not in the active contacts list.
4. Multi-domain orgs: each contact's email domain matches their legacy brand affiliation.
5. CRM-sourced contacts from Stage 1 have been validated externally (LinkedIn confirmation that they're still in post).
6. Persona reconciliation done: each contact's Stage 2 persona matches their validated scope.
7. Search budget: total searches ≤ 50.
8. Output file saved to `/mnt/user-data/outputs/{ORG_NAME}_Stage2_Validated_Contacts.md`.

---

## Handoff to Stage 3

The Stage 2 output is read by Stage 3, which:

- Composes opportunitySignals (using Stage 1 signals + Stage 2 named contacts as "Key Contacts").
- Composes the four-bin researchGaps from Stage 1 + Stage 2 raw gaps logs.
- Builds the final structured data payload.
- Saves contacts to Pipedriven via `save_org_people`.
- Publishes the intelligence report via `publish_org_intelligence`.
  Stage 2 does NOT call `save_org_people` or `publish_org_intelligence` directly. That's Stage 3.
