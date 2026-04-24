# Automated Bulletin Monitoring & Summarization

> **Platform:** Microsoft Power Automate (Cloud Flow) · **Tenant:** Microsoft 365 · **Audience:** IT / Security / Compliance

Automatically collects external IT and cybersecurity bulletin sources, creates dated intake files, and supports a human-in-the-loop Copilot summarization step — keeping all content inside the M365 tenant with no premium Power Automate connectors required.

> **Architecture note (Doc2 — license-safe revision):** The original design used the Power Automate HTTP connector and AI Builder "Create text with GPT." Both require premium licensing unavailable in a standard M365 + Copilot environment. This revised design uses **standard connectors only** for automation and relies on **manual Copilot prompting** (inside Word or OneDrive) for summarization and classification. Automation still handles ~70–80% of the effort — scheduling, source tracking, intake file creation, and audit logging.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Repository Structure](#repository-structure)
4. [Prerequisites](#prerequisites)
5. [Setup — Phase 1: SharePoint](#setup--phase-1-sharepoint)
6. [Setup — Phase 2: Build the Daily Collector Flow](#setup--phase-2-build-the-daily-collector-flow)
7. [Setup — Phase 2B: Manual Copilot Review](#setup--phase-2b-manual-copilot-review-it-staff)
8. [Setup — Phase 3: Build the Weekly Flow](#setup--phase-3-build-the-weekly-flow)
9. [Monitoring & Troubleshooting](#monitoring--troubleshooting)
10. [Adding or Removing Sources](#adding-or-removing-sources)
11. [Governance & Compliance](#governance--compliance)
12. [File Reference](#file-reference)

---

## Overview

This solution reduces manual bulletin monitoring effort for a Nova Scotia credit union IT and security team. Two scheduled Power Automate flows — one daily, one weekly — track approved external sources, create dated intake files, and maintain a complete audit log. A licensed IT staff member then opens each intake file, visits the source URL, and uses Copilot (inside Word or OneDrive) to summarise the content and apply a classification. The final bulletin is saved to Microsoft Teams. A Teams channel alert is posted manually only when the classification is **Attention Required**.

**What you gain:**

- Consistent daily and weekly source coverage with no missed days
- Scheduled reminders and intake files eliminate manual tracking
- Controlled, auditable Copilot output (same approved prompt every time)
- Human validation at every step — no AI tool making final decisions
- A complete audit trail inside M365 (SharePoint version history + Power Automate run history)
- Zero premium connector dependencies — works on standard M365 + Copilot licences

---

## Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│     PHASE 1 — Power Automate (Standard Connectors Only)           │
│                                                                   │
│  Scheduled Trigger (Daily / Weekly)                               │
│       │                                                           │
│       ▼                                                           │
│  Get items → Bulletin Sources (SharePoint List)                   │
│       │                                                           │
│       ▼  [Apply to each source]                                   │
│  Member source?                                                   │
│   Yes → Ingest email from shared mailbox OR flag Manual Review    │
│   No  →                                                           │
│       ▼                                                           │
│  Create intake file → SharePoint                                  │
│  (Teams > IT > Bulletins > Intake /                               │
│   YYYY-MM-DD – Source Name – Raw.txt)                             │
│       │                                                           │
│       ▼                                                           │
│  Log entry → Bulletin Run Log (SharePoint List)                   │
└───────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌───────────────────────────────────────────────────────────────────┐
│             PHASE 2 — IT Staff (Manual + Copilot)                 │
│                                                                   │
│  Open intake file → visit source URL → copy relevant text         │
│       │                                                           │
│       ▼                                                           │
│  Paste text into Word / OneDrive → ask Copilot to summarise       │
│  (using approved prompt from prompts/summarization_prompt.md)     │
│       │                                                           │
│       ▼                                                           │
│  Apply classification: Attention Required / Awareness Only        │
│       │                                                           │
│       ▼                                                           │
│  Paste into bulletin template → save to                           │
│  Teams > IT > Bulletins > Daily Monitoring                        │
│       │                                                           │
│       ▼  [Attention Required only]                                │
│  Post Teams message (link to bulletin file, one sentence)         │
└───────────────────────────────────────────────────────────────────┘
```

Everything stays inside the M365 tenant. No data is sent to third-party services. No premium Power Automate connectors are used.

---

## Repository Structure

```
.
├── README.md                        # This file — setup & instructions
├── Document.pdf                     # Original specification document
├── Doc2.pdf                         # License-safe architecture correction (supersedes original where noted)
├── config/
│   └── bulletin_sources.json        # Seed data for the Bulletin Sources SharePoint list
├── docs/
│   └── SOP.md                       # Standard Operating Procedure & governance notes
├── flow/
│   └── flow_overview.md             # Detailed Power Automate build reference
├── prompts/
│   └── summarization_prompt.md      # Approved Copilot prompt (use verbatim in manual review step)
└── templates/
    └── bulletin_template.md         # Markdown output template for bulletin files
```

---

## Prerequisites

Before you begin, confirm the following are available in your M365 tenant:

| Requirement | Details |
|-------------|---------|
| **Microsoft 365 licence** | Business Standard, E3, or E5 (Power Automate standard connectors are included) |
| **Power Automate access** | Ensure the account building the flow is a licensed Power Automate user — standard plan only; **no Premium plan required** |
| **Microsoft 365 Copilot** | Required for the manual summarization step (Phase 2); available in Word, OneDrive, and Teams |
| **SharePoint site** | An existing Teams IT SharePoint site with list-creation permissions |
| **Teams channel** | `Teams > IT` channel where bulletins will be stored and notifications posted |
| **Shared mailbox** | A monitored shared mailbox to receive FS-ISAC (member-only) digest emails |
| **Flow Owner account** | A service or user account that will own both flows and has permissions to SharePoint and Teams |

> **Connectors not required (and not used):** HTTP connector, AI Builder, Azure OpenAI, Copilot Studio, custom connectors, Dataverse. These are all premium and have been eliminated from the design.

---

## Setup — Phase 1: SharePoint

### 1.1 Create the Bulletin Sources List

1. Navigate to your Teams IT SharePoint site.
2. Select **New → List** and name it: **`Bulletin Sources`**
3. Add the following columns (in addition to the default `Title` column, which you can rename to `Source Name`):

| Column Name | Type | Required | Notes |
|-------------|------|----------|-------|
| Source Name | Single line of text | Yes | Human-readable display name |
| Source URL | Single line of text | Yes | Full `https://` URL |
| Source Type | Choice | Yes | Choices: `Public`, `Member`, `Signal` |
| Monitoring Frequency | Choice | Yes | Choices: `Daily`, `Weekly` |
| Enabled | Yes/No | Yes | Default: `Yes` — set to `No` to pause without deleting |
| Last Reviewed Hash | Single line of text | No | Leave blank; the flow writes this automatically |

### 1.2 Populate Initial Sources

Add the four initial sources from `config/bulletin_sources.json`:

| Source Name | Source URL | Type | Frequency |
|-------------|-----------|------|-----------|
| Canadian Centre for Cyber Security | https://www.cyber.gc.ca/en/cyber-security-alerts-advisories | Public | Daily |
| OSFI Technology & Cyber Guidance | https://www.osfi-bsif.gc.ca/en/guidance/guidance-library | Public | Daily |
| FS-ISAC Incident Response | https://www.fsisac.com/incident-response | Member | Daily |
| Downdetector Canada | https://downdetector.ca/ | Signal | Daily |

> **Note:** FS-ISAC is a member-only source. Do not configure the HTTP action for it. See [Step 3](#step-3--retrieve-source-content-http) for the email ingestion approach.

### 1.3 Create a Run Log List (Optional but Recommended)

Create a second SharePoint list named **`Bulletin Run Log`** with these columns:

| Column Name | Type |
|-------------|------|
| Source Name | Single line of text |
| Run Date | Date and time |
| Status | Choice (`Changed`, `No Change`, `Error`, `Manual Review`) |
| Notes | Multiple lines of text |

The flow writes one entry per source per run, giving you a complete audit trail.

### 1.4 Create the Bulletin Output Folders

In the Teams IT SharePoint document library (the one backing the **Files** tab in Teams):

1. Create folder: `Bulletins`
2. Inside `Bulletins`, create three sub-folders:
   - `Intake` *(raw intake files created by the flow — reviewed by IT staff)*
   - `Daily Monitoring`
   - `Weekly Monitoring`

---

## Setup — Phase 2: Build the Daily Collector Flow

Open [Power Automate](https://make.powerautomate.com) and follow these steps. All actions in this phase use **standard connectors only** — no premium licensing required.

### Step 1 — Create the Scheduled Flow

1. Select **Create → Scheduled cloud flow**.
2. Set the following:
   - **Name:** `Daily – External IT & Cyber Bulletin Collector`
   - **Starting:** Today's date
   - **Repeat every:** `1 Day`
   - **At:** `07:00 AM` Atlantic Standard Time (11:00 UTC / 12:00 UTC during ADT)
3. Click **Create**.

### Step 2 — Retrieve Sources from SharePoint

Add the first action inside the flow:

1. **Action:** `Get items` (SharePoint)
   - **Site Address:** Select your Teams IT SharePoint site
   - **List Name:** `Bulletin Sources`
   - **Filter Query:** `Enabled eq 1`
   - **Top Count:** `100` (increase if you add many sources)

2. Add action: **Apply to each**
   - **Select an output from previous steps:** `value` (output of Get items)

All remaining steps in this section are built **inside** the Apply to each loop.

### Step 3 — Handle Source Type and Create Intake File

Inside the loop, add a **Condition** to check the source type:

- **Condition:** `@{items('Apply_to_each')?['Source_x0020_Type']}` **is not equal to** `Member`

**If Yes (Public or Signal source):**

1. Add action: **Create file** (SharePoint)
   - **Site Address:** Teams IT SharePoint site
   - **Folder Path:** `/Bulletins/Intake`
   - **File Name:**
     ```
     @{formatDateTime(utcNow(), 'yyyy-MM-dd')} – @{items('Apply_to_each')?['Source_x0020_Name']} – Raw.txt
     ```
   - **File Content:**
     ```
     Source: @{items('Apply_to_each')?['Source_x0020_Name']}
     URL: @{items('Apply_to_each')?['Source_x0020_URL']}
     Date: @{formatDateTime(utcNow(), 'yyyy-MM-dd')}

     ACTION REQUIRED: Open the URL above, copy relevant content, then use Copilot to summarise.
     Use the approved prompt in prompts/summarization_prompt.md.
     Save the final bulletin to Bulletins/Daily Monitoring.
     ```
   - Rename this action to `CreateIntakeFile`.

2. Add action: **Create item** (SharePoint — Bulletin Run Log list)
   - Source Name: `@{items('Apply_to_each')?['Source_x0020_Name']}`
   - Run Date: `@{utcNow()}`
   - Status: `Intake Created`
   - Notes: `Intake file created. Awaiting manual Copilot review.`

**If No (Member source — e.g., FS-ISAC):**

Choose one of the following approaches:

- **Email ingestion approach:**
  1. Add action: **Get emails (V3)** (Office 365 Outlook)
     - **Mailbox Address:** your monitored shared mailbox address
     - **Folder:** Inbox
     - **Filter:** `From` contains `fsisac.com`
     - **Top:** `5`
  2. Add action: **Create file** (SharePoint) to save the email body as the intake file content (same folder path and naming convention as above).

- **Manual flag approach:**
  1. Add action: **Create file** (SharePoint) with content: `"Manual review required — FS-ISAC content must be reviewed directly at https://www.fsisac.com."`
  2. Add action: **Create item** (SharePoint) to write a `Manual Review` status to the Run Log list.

---

## Setup — Phase 2B: Manual Copilot Review (IT Staff)

After the flow runs each morning, an IT staff member performs the following steps for each intake file created.

> This step uses **Microsoft 365 Copilot** inside Word or OneDrive — no additional licensing beyond your M365 Copilot subscription is required.

### Step A — Open the Intake File

1. Navigate to `Teams > IT > Files > Bulletins > Intake`.
2. Open today's intake file (e.g., `2025-01-15 – Canadian Centre for Cyber Security – Raw.txt`).
3. Open the source URL listed in the file in a browser tab.

### Step B — Copy and Summarise with Copilot

1. Copy the relevant content from the source URL.
2. Open a new Word document or OneDrive file.
3. Paste the copied content.
4. Select all pasted text, then use Copilot and paste the **full approved prompt verbatim** (from `prompts/summarization_prompt.md`):

   ```
   Summarize the following content for a Canadian credit union IT and security audience.

   Requirements:
   - Professional tone
   - No emojis or marketing language
   - Focus on operational relevance
   - Indicate whether immediate action is required
   - Note potential branch impact
   - Use short paragraphs or bullet points
   - Do not speculate

   End the summary with:
   "Source: [URL]"
   ```

5. Copilot returns a structured summary. Review it for accuracy — do **not** accept it without reading it.

### Step C — Classify and Create the Final Bulletin

1. Apply classification based on the summary content:
   - **Attention Required** — if the summary indicates immediate action, active exploitation, service disruption, or a critical vulnerability
   - **Awareness Only** — all other cases

2. Open `templates/bulletin_template.md` and fill in the placeholders.

3. Save the completed bulletin to `Teams > IT > Bulletins > Daily Monitoring` using the file name format:
   ```
   YYYY-MM-DD – Daily IT & Cyber Watch.md
   ```

4. If classification is **Attention Required**, post a message in the `Teams > IT` channel:
   ```
   ⚠️ Attention Required – new bulletin posted for [Source Name].
   Review: [link to SharePoint bulletin file]
   ```
   Do **not** paste the full summary into the Teams message.

---

## Setup — Phase 3: Build the Weekly Flow

1. In Power Automate, open the daily collector flow.
2. Select the **⋯ menu → Save As**.
3. Name the copy: `Weekly – External IT & Cyber Bulletin Collector`
4. Make the following changes:
   - **Recurrence:** Change frequency to `Week`, interval `1`.
   - **Intake folder path (Step 3):** Change to `/Bulletins/Intake` with a weekly prefix in the file name (e.g., prepend `Weekly – `).
5. Save and turn on the flow.

For the manual Copilot review step (Phase 2B), use the same process but append the following to your Copilot prompt:
```
Focus on trends and recurring risks rather than individual daily alerts.
```

Save weekly bulletins to `Teams > IT > Bulletins > Weekly Monitoring`.

---

## Monitoring & Troubleshooting

### Normal Operations

- Check **Power Automate Run History** daily for the first two weeks after go-live.
- Go to **My Flows → Daily – External IT & Cyber Bulletin Collector → Run history**.
- Each run should complete with status `Succeeded`. Review any `Failed` runs immediately.

### Setting Up Failure Alerts

1. Open the flow.
2. Select **⋯ → Settings**.
3. Enable **Send run failure email** and confirm the recipient address.

### Common Failure Causes

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| SharePoint "Create file" action fails | Permissions, column name mismatch, or folder path not found | Verify the flow owner account has Contribute access; confirm the `Bulletins/Intake` folder exists |
| "Get emails" action fails (Member sources) | Shared mailbox permissions changed or filter too narrow | Check shared mailbox access; adjust the sender filter |
| SharePoint "Get items" fails | List name changed or site URL moved | Update the action's Site Address and List Name fields |
| Flow times out | Too many sources running serially | Split large source lists into separate flows |

### Manual Fallback

If the flow fails and cannot be restored the same day:

1. Visit each enabled source URL directly (use `config/bulletin_sources.json` as the source list).
2. Copy relevant content and use Copilot to summarise using the approved prompt.
3. Fill in `templates/bulletin_template.md` and save manually to `Teams > IT > Bulletins > Daily Monitoring`.
4. Log the manual run in the Bulletin Run Log SharePoint list.
5. Notify the Flow Owner to restore automated operation within one business day.

---

## Adding or Removing Sources

All source management is done through the **Bulletin Sources** SharePoint list — no flow changes required.

**To add a source:**
1. Get approval from the IT Security Lead.
2. Add a new item to the Bulletin Sources list with all required columns.
3. Set `Enabled` to `Yes`.
4. Update `config/bulletin_sources.json` in this repository to keep the seed file current.

**To pause a source:**
1. Find the source in the Bulletin Sources list.
2. Set `Enabled` to `No`.
3. The flow will skip it on all subsequent runs.

**To permanently remove a source:**
1. Get approval from the IT Security Lead.
2. Set `Enabled` to `No` first (for a clean cutover).
3. Delete the list item after confirming no in-flight bulletins reference it.
4. Remove the entry from `config/bulletin_sources.json` and commit the change.

> Member-only sources (Source Type = `Member`) require additional setup for email ingestion. Coordinate with the Flow Owner before adding new member-only sources.

---

## Governance & Compliance

### Governance Statement

> **Copilot-assisted summaries are used for awareness only.**  
> Final incident classification, escalation, and regulatory reporting decisions remain the responsibility of the Credit Union.

This statement appears in every bulletin. It must not be removed from the template. Human review of every Copilot output is required before any classification is applied.

### Roles

| Role | Responsibility |
|------|----------------|
| Power Automate Flow Owner | Build, maintain, and monitor flows; update source list; resolve failures |
| IT Security Lead | Review all Attention Required bulletins; determine escalation; approve source changes |
| Compliance Officer | Annual SOP review; oversight of regulatory reporting obligations |
| Branch Staff | No direct interaction with the system; receive escalated guidance from IT only |

### Escalation Path

```
Bulletin classified as Attention Required
  └─► IT Security Lead reviews full bulletin in SharePoint
        └─► Is a confirmed threat or action item?
              ├─ Yes → Notify affected teams; open incident ticket if warranted
              │         Regulatory matters → Compliance Officer
              └─ No  → Log as reviewed; no further action required
```

### Audit Trail

The following artefacts are retained automatically:

- **Power Automate Run History** — every flow execution, duration, and step-level success/failure
- **SharePoint Bulletin Files** — all generated Markdown bulletins with version history
- **Bulletin Run Log list** — per-source per-run status entries
- **Last Reviewed Hash column** — evidence of what content existed at each review

No bulletin file is to be deleted. Files are archived in place in SharePoint.

### SOP

The full Standard Operating Procedure (including annual review checklist) is in [`docs/SOP.md`](docs/SOP.md).

---

## File Reference

| File | Purpose |
|------|---------|
| `README.md` | This document — full setup instructions |
| `Document.pdf` | Original specification document |
| `Doc2.pdf` | License-safe architecture correction (supersedes original where noted) |
| `config/bulletin_sources.json` | Seed data for the Bulletin Sources SharePoint list |
| `docs/SOP.md` | Standard Operating Procedure, roles, escalation, audit |
| `flow/flow_overview.md` | Condensed Power Automate build reference |
| `prompts/summarization_prompt.md` | Approved Copilot prompt for manual summarization (use verbatim) |
| `templates/bulletin_template.md` | Markdown template for completed bulletin files |

---

## Flows Summary

| Flow Name | Schedule | Output Folder |
|-----------|----------|---------------|
| Daily – External IT & Cyber Bulletin Collector | Daily @ 07:00 AST | `Teams > IT > Bulletins > Intake` (then manually to `Daily Monitoring`) |
| Weekly – External IT & Cyber Bulletin Collector | Weekly | `Teams > IT > Bulletins > Intake` (then manually to `Weekly Monitoring`) |
