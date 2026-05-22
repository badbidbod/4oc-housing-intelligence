---
name: hms-ha-intelligence-stage3
description: "Stage 3 of HMS housing association intelligence. Takes Stage 1 Research Notes and Stage 2 Validated Contacts, composes the final structured intelligence payload across seven Pipedriven sections, saves validated contacts to Pipedriven, and publishes the intelligence report via publish_org_intelligence. Composes opportunitySignals from Stage 1 procurement triggers and Stage 2 outreach signals plus Key Contacts. Composes researchGaps as a four-bin categorisation (Search-actionable, Outreach-only, Structural unknown, Resolved during research). Final stage before Pipedriven publication. No new research; composition and publication only. Use after Stage 1 and Stage 2 have produced their output files. Trigger on HA HMS intelligence Stage 3, publish HA intelligence to Pipedriven, compose opportunitySignals, save HA contacts to Pipedriven, four-bin research gaps, HA intelligence final report."
---

# HA Skill, Stage 3: Profile Assembly and Pipedriven Publish

**Skill.** `hms-ha-intelligence-stage3`, Stage 3 of 3.

**Status.** Draft, 21 May 2026. Companion to Tag Schema v0.4, HA Three-Stage Architecture v0.3, and the Stage 1 and Stage 2 skills.

**Purpose.** Take the Stage 1 Research Notes file and the Stage 2 Validated Contacts file, compose the final structured intelligence payload across the seven Pipedriven sections, save validated contacts to Pipedriven, and publish the intelligence report via `publish_org_intelligence`.

**Scope, in.** Composition (assembling section content from Stage 1 and Stage 2 outputs), opportunitySignals authoring, researchGaps four-bin categorisation, markdown report drafting, Pipedriven contact save, Pipedriven intelligence publish.

**Scope, out.** New research (Stage 1's job). People validation (Stage 2's job). Any web search beyond resolving the Pipedriven organizationId.

---

## Operating mode

Single conversation per org. No new web research. Pipedriven MCP active throughout.

**Execute inline in this conversation. DO NOT dispatch to `launch_extended_search_task` or any research subagent.** Stage 3 is composition, not research. If a gap surfaces that needs more research, route it to the researchGaps four-bin (not to a new search) and proceed. Deep research should be disabled at the UI level when this skill runs.

---

## Inputs

1. **Stage 1 Research Notes file** for the org. Standard locations to check, in order:
   - `/mnt/user-data/uploads/{ORG_NAME}_Stage1_Research_Notes.md`
   - `/mnt/project/{ORG_NAME}_Stage1_Research_Notes.md`
   - `/mnt/user-data/outputs/{ORG_NAME}_Stage1_Research_Notes.md`
2. **Stage 2 Validated Contacts file** for the org. Standard locations to check, in order:
   - `/mnt/user-data/uploads/{ORG_NAME}_Stage2_Validated_Contacts.md`
   - `/mnt/project/{ORG_NAME}_Stage2_Validated_Contacts.md`
   - `/mnt/user-data/outputs/{ORG_NAME}_Stage2_Validated_Contacts.md`
3. **Pipedriven organisation record** for the org (resolved via `search_organizations`).
   If either Stage 1 or Stage 2 file is missing, stop and ask the user. Do not proceed without both.

---

## Outputs

1. **Local report file**: `{ORG_NAME}_Stage3_Pipedriven_Report.md` saved to `/mnt/user-data/outputs/`. Human-readable artifact retained alongside Stages 1 and 2.
2. **Pipedriven contacts saved** via `save_org_people` (one record per active contact from Stage 2).
3. **Pipedriven intelligence report published** via `publish_org_intelligence` (with structuredData payload + markdownContent + title + reportDate).

---

## Core concepts

**No new research.** Stage 3 does not search the web. All content comes from Stage 1 and Stage 2 files. If a gap is identified during composition, route to the researchGaps four-bin.

**Pipedriven org resolution first.** Before any publishing, resolve the org in Pipedriven via `search_organizations`. Capture the organizationId. If the org doesn't exist, call `create_organization` with sector HOUSING. If multiple near-matches return, ask the user before proceeding.

**Section composition rules.** Five sections (overview, strategicDirection, corporateRisks, challenges, financialHealth) lift directly from Stage 1 tag blocks with minor formatting. Two sections (opportunitySignals, researchGaps) are authored fresh in Stage 3, synthesising Stage 1 + Stage 2 inputs.

**opportunitySignals authoring.** This is the BD action layer. It combines: Stage 1 procurement signals + Stage 2 key outreach signals + Stage 2 validated Key Contacts + a 4OC angle. The output frames the commercial conversation, not the research narrative.

**researchGaps four-bin discipline.** Every gap from Stage 1 raw gaps log and Stage 2 raw gaps log is categorised into one of four bins:

- **Search-actionable**: a clear next search or document fetch would likely close this. Example: "Bond prospectus not reviewed; fetch from RNS."
- **Outreach-only**: cannot be resolved by external research. Only direct contact can close it. Example: "Current internal contract end date for Aareon ActiveH."
- **Structural unknown**: the org itself hasn't decided or published. Example: "Post-merger HMS rationalisation plan; board-level discussion not yet public."
- **Resolved during research**: closed during Stage 1 or Stage 2; recorded for completeness.
  **Markdown report = published artifact.** The markdown content published to Pipedriven is what humans see when they query the org. Write it as a standalone document: someone reading it without context should understand the position.

**Save contacts before publishing intelligence.** Order matters: `save_org_people` first (so contacts exist in Pipedriven for the Key Contacts links to resolve), then `publish_org_intelligence`.

---

## Steps

### Step S1. Load Stage 1 and Stage 2 outputs

_Purpose._ Read both input files. Extract org name, all tag blocks, signals, contacts, gaps logs.

_Process._

1. Locate Stage 1 file in standard locations or via user-provided path.
2. Locate Stage 2 file similarly.
3. Read both files in full.
4. Confirm both files are for the same org (name match).
5. If either missing, stop and ask the user.

### Step S2. Resolve organisation in Pipedriven

_Purpose._ Get the organizationId needed for publishing.

_Process._

1. Call `search_organizations` with the org name (and legacy brand names if post-merger).
2. If one clear match: capture organizationId.
3. If multiple near-matches: ask the user which one before proceeding.
4. If no match: call `create_organization` with sector HOUSING, country UK, and the org's website. Capture the returned organizationId.
   _Discipline._

- Do NOT guess on a near-match. A wrong organizationId publishes intelligence to the wrong org record.
- For post-merger orgs, the current legal entity is the target. Legacy brands are not the publish destination.

### Step S3. Compose the five lifted sections

_Purpose._ Pull `overview`, `strategicDirection`, `corporateRisks`, `challenges`, `financialHealth` from Stage 1 into the structuredData object. Minor formatting only; no rewriting.

_Per section:_

1. Read the Stage 1 section block.
2. Lift the tag values into structured fields. Each tag becomes a key-value pair in the structured object.
3. Preserve the Evidence notes paragraph(s) at the end of each section block.
4. Apply any Stage 2 contextual additions (e.g., Stage 2 may surface executive churn signals that belong in strategicDirection; if so, append to the section without rewriting Stage 1's content).
   _Structured data format per section (illustrative for overview):_

```json
"overview": {
  "organisationType": "Housing Association (registered provider, charitable)",
  "region": "London",
  "stockSize": "~67,000 homes",
  "hmsSupplier": "Aareon (inferred)",
  "hmsProduct": "Northgate (inferred)",
  "hmsContractStartDate": "Unknown",
  "hmsContractEndDate": "Unknown",
  "hmsContractStatus": "Unknown",
  "contractLifecycleStage": "Unknown",
  "crmWarmness": "3 (John Carey)",
  "accountOwner": "Unknown from CRM",
  "evidenceNotes": "..."
}
```

Repeat for strategicDirection, corporateRisks, challenges, financialHealth. Tag names match the v0.4 schema.

### Step S4. Compose opportunitySignals

_Purpose._ Author the BD action layer. This is the most synthetic section; it converts Stage 1 + Stage 2 signals into a commercial conversation. Composition, not capture.

_Source material:_

- Stage 1 corporateRisks, challenges, strategicDirection, financialHealth (the pressure picture)
- Stage 1 Procurement signals (timing)
- Stage 1 Competitor presence (competitive context)
- Stage 2 Key outreach signals
- Stage 2 Validated contacts (becomes Key Contacts)
  _Composition steps:_

1. **Identify the primary Trigger.** Choose one (compound if two are live):
   - Contract-status (HMS contract expiring or recently renewed)
   - Vendor M&A (Civica/Blackstone, MRI/Capita, Aareon/MIS Active, Aareon/HomeMaster)
   - Performance-led (RSH downgrade, Ombudsman findings, TSM decline)
   - Regulatory recovery (active voluntary undertaking, compliance plan)
   - Leadership change (new CEO / CIO / CCO)
   - Merger (recent or imminent)
   - Financial pressure (downgrade, deficit, bond suspension)
2. **State urgency** with one-line reason:
   - HIGH = trigger is live now (contract expiring within 12 months; active RSH recovery; M&A forcing decision; consumer inspection imminent)
   - MEDIUM = trigger present, no forced decision point (18-36 months out)
   - LOW = monitor only
3. **Compose pressure points.** 3-5 short statements of what the org's leadership is dealing with right now. Each one ties a Stage 1 signal to its commercial implication for them (not for 4OC). Example: "V1-to-V2 regrade citing capital programme debt servicing; need to demonstrate operational efficiency without cutting investment programme."
4. **Compose entry points.** 3-5 composed reasons to engage. Each = [their pressure] + [matched 4OC service line] + [named proof point if known] + [why now] + [conversation starter / contact]. NOT a data point on its own. Match against the 4OC service catalogue (six categories: Digital Data and Technology; Service Design and Improvement; Project and Programme Management; Change Management; Strategy and Organisation Design; Leadership Culture and Capability). All service lines are on the table - the bid-facing/relationship-led distinction only applies to cold tender prospects. Where you can credibly name a 4OC proof point (named client engagement and outcome), include it; if not, name only the service line.
5. **Compose signal coherence narrative.** One short paragraph synthesising what the org's leadership is actually dealing with. Not a list of signals; a read of how they connect. Cite the 2-3 most material signals by name. This is what a senior BD person would answer to "what's going on at X?"
6. **State 4OC angle.** One-line summary tying the entry points to system-led (HMS procurement, requirements, data migration) and/or non-system-led (operating model, process, capability) framing. Follows from the entry points.
7. **List Key Contacts** from Stage 2 validated contacts. Persona structure (Strategic Lead, Operational Leads per function, Technical Lead, Board, Other). Each contact MUST have a one-line rationale stating WHY this person for the BD conversation (not their job description). Example: "Owns the 44% complaints TSM function; 17+ years institutional knowledge; route into Customer & Communities executive layer." Names with emails inline.
8. **Recommend approach.** One line: named contact + specific action + timing + why now. Example: "Approach Sarah Roxby within 4 weeks with a one-pager on complaints process redesign tied to her 44% TSM and the pending consumer inspection."
9. **Recommend outreach sequence.** Who first, who second, who later. Justify ordering by decision-making influence on the BD topic, not by warmness alone.
10. **Note known competitors.** From Stage 1 Competitor presence. If 4OC is on a relevant framework the competitors aren't (or vice versa), state it.
    _Structured data format:_

```json
"opportunitySignals": {
  "primaryTrigger": "Regulatory recovery + Leadership change",
  "urgency": "HIGH - active RSH voluntary undertaking; new CCO March 2026 rebuilding team",
  "pressurePoints": [
    "V1-to-V2 regrade citing capital programme debt servicing; need to demonstrate operational efficiency without cutting investment",
    "44% complaints TSM (lowest of 12 measures); consumer standards inspection not yet conducted",
    "Core HMS supplier unconfirmed beneath fragmented stack (Capita, Salesforce, SAP); MRI/Capita acquisition exposure unclear"
  ],
  "entryPoints": [
    {
      "pressure": "44% complaints TSM ahead of pending consumer inspection",
      "fourOCFit": "Service Review + Business Process Design",
      "proofPoint": "Birmingham Root and Branch Housing Service Review (56 recommendations, complaints process improvements); ForHousing Rapid Improvement Cycles for damp and disrepair",
      "whyNow": "Inspection not yet conducted; new CEO programme creates appetite",
      "conversationStarter": "Sarah Roxby owns the function; Birmingham and ForHousing are direct comparators"
    },
    {
      "pressure": "V2 regrade citing capital programme debt servicing",
      "fourOCFit": "Organisational Review + Target Operating Model Design",
      "proofPoint": "Rotherham TOM Design post-commissioner intervention; Genesis Communities TOM Redesign (£726k annual cost reduction); Birmingham HRA Review (£5.7m additional annual income)",
      "whyNow": "CEO in first 18 months; named 3-year programme launched April 2025",
      "conversationStarter": "Martyn Shaw or Sarah Cooper; Rotherham and Birmingham HRA are direct LG-housing comparators"
    },
    {
      "pressure": "Core HMS supplier unconfirmed + Capita/MRI M&A exposure",
      "fourOCFit": "System & Technology Review + Digital Discovery + Requirements Gathering",
      "proofPoint": "Thames Valley Housing finance discovery and requirements gathering; Soho Housing finance system replacement; Birmingham ICT Discovery Project",
      "whyNow": "MRI/Capita acquisition forces decision review; V2 efficiency pressure makes the question commercial",
      "conversationStarter": "Simon Bastow (Head of Business Systems); TVH and Soho are direct HA system comparators"
    }
  ],
  "signalCoherence": "Vico's leadership is running a named 3-year transformation programme under a new CEO while absorbing a V2 regrade that ties capital ambition to financial fragility. The complaints handling failure (44% TSM, lowest of 12 measures) is the sharpest operational signal and sits ahead of a consumer standards inspection that hasn't happened yet. Beneath both, the core HMS estate is unconfirmed and the Capita/MRI acquisition adds vendor uncertainty. The picture is coherent: an organisation reaching for growth while needing to demonstrate operational efficiency and pass the next inspection.",
  "fourOCAngle": "Non-system-led primary: operating model and process review under V2 pressure; complaints redesign before inspection. System-led secondary: HMS discovery via Simon Bastow.",
  "keyContacts": [
    {"name": "Sarah Roxby", "role": "Executive Director, Customer and Communities", "persona": "Strategic Lead - PRIMARY", "email": "sarahroxby@vicohomes.co.uk", "emailStatus": "Inferred (medium)", "rationale": "Owns the 44% complaints TSM function; 17+ years institutional knowledge; route into Customer & Communities executive layer"},
    {"name": "Simon Bastow", "role": "Head of Business Systems", "persona": "Technical Lead", "email": "simonbastow@vicohomes.co.uk", "emailStatus": "Inferred (medium)", "rationale": "Only credible route to HMS supplier and contract position; knows the system pain points"},
    {"name": "Martyn Shaw", "role": "Chief Executive", "persona": "Executive (CEO)", "email": "martynshaw@vicohomes.co.uk", "emailStatus": "Inferred (medium)", "rationale": "First 18 months in post; technical services background; escalation target once director relationship established"}
  ],
  "recommendedApproach": "Approach Sarah Roxby within 4 weeks with a one-pager on complaints process redesign tied to her 44% TSM and pending consumer inspection; reference Birmingham Root and Branch and ForHousing as comparators",
  "recommendedOutreachSequence": [
    "1. Sarah Roxby (owns complaints function; sharpest live pressure point)",
    "2. Simon Bastow (HMS discovery; parallel track)",
    "3. Martyn Shaw (escalation once director relationship established)"
  ],
  "knownCompetitors": "None evidenced at Vico Homes in Competitor Research Log.",
  "evidenceNotes": "V2 regrade: RSH judgement 14 January 2026. 44% complaints: WDH TSM Survey 2024. New CEO: Inside Housing January 2025. HMS gap: 53 procurement notices past year, none for HMS."
}
```

### Step S5. Compose researchGaps (four-bin)

_Purpose._ Categorise every gap from Stage 1 raw gaps log and Stage 2 raw gaps log into one of four bins.

_Process:_

1. Combine Stage 1 raw gaps log and Stage 2 raw gaps log.
2. For each gap, choose the bin based on what would close it:
   - Can a specific document fetch or targeted search close it? → **Search-actionable**
   - Does it need direct contact with the org? → **Outreach-only**
   - Has the org itself not decided/published? → **Structural unknown**
   - Was it closed during research? → **Resolved during research**
3. For each gap, note the recommended next action (where applicable).
   _Structured data format:_

```json
"researchGaps": {
  "searchActionable": [
    {"gap": "Bond prospectus tech-risk language", "nextAction": "Fetch Sanctuary Capital PLC Programme Admission Particulars March 2026 from RNS"},
    {"gap": "Precise Ombudsman maladministration count", "nextAction": "Direct fetch of housing-ombudsman.org.uk landlord profile page"}
  ],
  "outreachOnly": [
    {"gap": "Bromford/Aareon ActiveH current contract end date", "nextAction": "Direct contact with John Carey or NHG procurement"},
    {"gap": "LiveWest HMS supplier and product", "nextAction": "Outreach via warm contact"}
  ],
  "structuralUnknown": [
    {"gap": "Post-merger HMS rationalisation plan for BFL", "nextAction": "Likely a board-level decision not yet taken; monitor for public signal"},
    {"gap": "S/4HANA migration approach at Sanctuary", "nextAction": "Internal decision pending; track for procurement notice"}
  ],
  "resolvedDuringResearch": [
    {"gap": "John Carey role (Stage 1 unknown)", "resolution": "Head of Procurement, confirmed via RocketReach + CRM (Stage 2)"},
    {"gap": "Adam Cresser title (Stage 1 'Data & AI Director')", "resolution": "Updated to 'Director of Data, Analytics & Digital' (Stage 2 LinkedIn)"}
  ]
}
```

_Discipline._

- Every gap from Stage 1 and Stage 2 raw gaps logs must appear in one of the four bins. Count before publishing.
- Avoid Search-actionable items that are 4+ search attempts in (those graduate to Outreach-only or Structural unknown).
- Outreach-only items are also the highest-value Stage 3 outputs for BD: they tell the team what they can't know without picking up the phone.

### Step S6. Compose the markdown report

_Purpose._ Author the human-readable artifact that gets published as `markdownContent` and saved locally.

_Structure:_ See output template below.

_Style:_

- Plain English; peer-to-peer tone (the audience is internal 4OC team and the Pipedriven user querying the record).
- No em-dashes (use commas, colons, semicolons, or single hyphens).
- No fillers ("we reached out", "compare notes", "landed", "runway").
- Apply the "SO WHAT?" test: every paragraph should answer why the reader should care.
- Tight: high-signal over comprehensive. Aim for ~1,200-1,800 words across the whole report.

### Step S7. Save contacts to Pipedriven

_Purpose._ Push each Stage 2 validated active contact into Pipedriven via `save_org_people` (or the equivalent contact-save tool; look up via tool_search if not visible).

_Per contact (active contacts only, not departures):_

- Name
- Role / title (validated in Stage 2)
- Email
- Persona category
- Email status flag (Verified / Inferred high / Inferred medium / Inferred best-guess) carried as a note
  _Departures are NOT saved as active contacts._ They are noted in the markdown report (in opportunitySignals or a separate departures note) but not pushed to Pipedriven as live contacts.

_Discipline._

- Order: save contacts BEFORE publishing intelligence (Key Contacts links in the intelligence report should resolve to existing Pipedriven contact records).
- If `save_org_people` returns errors for specific contacts, capture in a Stage 3 publish log and continue with the rest. Don't block the publish on a single failed contact save.

### Step S8. Publish intelligence to Pipedriven

_Purpose._ Call `publish_org_intelligence` with the composed payload.

_Call structure:_

```
publish_org_intelligence(
  organizationId: <from S2>,
  title: "{Org name} HMS Performance Intelligence Profile",
  reportDate: "{YYYY-MM-DD}",
  structuredData: { <7 sections from S3 + S4 + S5> },
  markdownContent: "<full markdown from S6>"
)
```

_Discipline._

- All 7 sections present in structuredData (overview, strategicDirection, corporateRisks, challenges, financialHealth, opportunitySignals, researchGaps). Missing sections are a publish defect; the tool will accept the call but the published record will be incomplete.
- Title format: "{Org name} HMS Performance Intelligence Profile".
- reportDate is today's date in ISO format.

### Step S9. Verify and report back

_Purpose._ Confirm the publish succeeded and summarise what was done.

_Process:_

1. Capture the response from `publish_org_intelligence` (article ID, status).
2. Save the local report file to `/mnt/user-data/outputs/{ORG_NAME}_Stage3_Pipedriven_Report.md`.
3. Summarise to the user:
   - Org resolved to organizationId
   - Contacts saved (count, with any failures noted)
   - Intelligence published (article ID)
   - Outreach sequence recommended
   - Top 3 researchGaps Outreach-only items (these are the BD team's priority follow-ups)

---

## Output file template (local markdown artifact and publishable markdownContent)

```markdown
# {Org name} HMS Performance Intelligence Profile

**Report date.** {YYYY-MM-DD}
**Author.** 4OC, via hms-ha-intelligence Stages 1-3
**Status.** Published to Pipedriven

---

## Executive summary

(2-4 sentences. The primary Trigger, the 4OC angle, the warmest contact, the most material gap.)

---

## Overview

(Lifted from Stage 1 overview section. Tag block plus evidence notes.)

---

## Strategic direction

(Lifted from Stage 1 strategicDirection section, with any Stage 2 additions appended.)

---

## Corporate risks

(Lifted from Stage 1 corporateRisks section.)

---

## Challenges

(Lifted from Stage 1 challenges section.)

---

## Financial health

(Lifted from Stage 1 financialHealth section.)

---

## Opportunity signals

**Primary trigger.** {Trigger; compound if applicable}

**Urgency.** {HIGH/MEDIUM/LOW with one-line reason}

**Pressure points.**

- {Pressure point 1}
- {Pressure point 2}
- {Pressure point 3}

**Entry points.**

1. **{Pressure short title}.** {4OC fit service lines}. Proof: {named proof points}. Why now: {timing}. Conversation starter: {contact + comparator}.
2. ...
3. ...

**Signal coherence.** {One short paragraph synthesising what the org's leadership is dealing with right now and how the signals connect.}

**4OC angle.** {One-line summary tying entry points to system-led / non-system-led framing}

**Key contacts.**

| Name | Role | Persona | Email | Email status | Rationale |
| ---- | ---- | ------- | ----- | ------------ | --------- |
| ...  | ...  | ...     | ...   | ...          | ...       |

**Recommended approach.** {Named contact + specific action + timing + why now}

**Recommended outreach sequence.**

1. {Contact} ({reason})
2. {Contact} ({reason})
3. {Contact} ({reason})

**Known competitor presence.** {From Stage 1 Competitor presence}

**Evidence notes.** {Source pointers for the trigger, urgency, and entry points}

---

## Research gaps

**Search-actionable.** (Gaps closable by a specific next search or document fetch.)

- {Gap}: {next action}
- ...

**Outreach-only.** (Gaps that need direct contact with the org. These are the BD team's priority follow-ups.)

- {Gap}: {next action}
- ...

**Structural unknown.** (Gaps where the org itself hasn't decided or published; monitor for public signal.)

- {Gap}: {monitoring note}
- ...

**Resolved during research.** (Gaps closed during Stage 1 or Stage 2.)

- {Gap}: {resolution}
- ...

---

## Sources

(From Stage 1 Sources + Stage 2 Sources, deduplicated, tier-coded.)
```

---

## Quality checks before publishing

1. Both Stage 1 and Stage 2 input files were read in full.
2. Organizatio_id resolved via Pipedriven (not assumed).
3. All seven structuredData sections present and populated. Count: 7.
4. opportunitySignals has all required fields: primaryTrigger, urgency, pressurePoints (3-5), entryPoints (3-5 objects, each with pressure/fourOCFit/proofPoint/whyNow/conversationStarter), signalCoherence, fourOCAngle, keyContacts (with email AND rationale), recommendedApproach, recommendedOutreachSequence, knownCompetitors, evidenceNotes.
5. researchGaps has all four bins populated (any may be empty arrays, but the structure must be present).
6. Every Stage 2 active contact appears in Key Contacts table with an email and status flag.
7. Departures are NOT in Key Contacts. They may be noted in the markdown but not in structuredData keyContacts.
8. Markdown report passes "SO WHAT?" test: every paragraph earns its place.
9. No em-dashes in the markdown report.
10. Local file saved to `/mnt/user-data/outputs/{ORG_NAME}_Stage3_Pipedriven_Report.md`.
11. `save_org_people` called BEFORE `publish_org_intelligence`.
12. `publish_org_intelligence` call returned a success status; article ID captured.

---

## Failure modes and recovery

- **Stage 1 or Stage 2 file missing.** Stop. Ask the user. Do not generate partial publishes.
- **Org not in Pipedriven.** Use `create_organization` with sector HOUSING. Capture the new organizationId and proceed.
- **Org has near-matches in Pipedriven.** Ask the user before proceeding. Wrong organizationId = publish to wrong record.
- **`save_org_people` fails for one contact.** Capture in log, continue with the rest. The intelligence publish still proceeds; the failed contact is flagged in the Step S9 summary.
- **`publish_org_intelligence` fails.** Capture the error, save the local report file anyway (so the work isn't lost), and report the publish failure with the error to the user.

---

## Handoff back to the process

After Stage 3:

- The intelligence record exists in Pipedriven, queryable via `get_org_intelligence`.
- Contacts exist in Pipedriven, available for outreach via the recommended sequence.
- Researchgaps Outreach-only items are the BD team's call list.
- If new signals emerge later (regulatory action, contract notice, leadership move), the cycle returns to Stage 1 to re-research and Stage 3 to re-publish (the publish tool is idempotent on date, so re-running the same day updates; re-running a different day archives the prior version).
