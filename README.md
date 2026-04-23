# Automated Bulletin Monitoring & Summarization

> **Platform:** Microsoft Power Automate (Cloud Flow) · **Tenant:** Microsoft 365 · **Audience:** IT / Security / Compliance

Automatically monitors external IT and cybersecurity bulletin sources, detects new content, summarises it using Copilot / AI Builder, and saves structured Markdown reports directly inside Microsoft Teams — with no manual link-checking and no AI output leaving the M365 tenant.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Repository Structure](#repository-structure)
4. [Prerequisites](#prerequisites)
5. [Setup — Phase 1: SharePoint](#setup--phase-1-sharepoint)
6. [Setup — Phase 2: Build the Daily Flow](#setup--phase-2-build-the-daily-flow)
   - [Step 1 — Create the Scheduled Flow](#step-1--create-the-scheduled-flow)
   - [Step 2 — Retrieve Sources from SharePoint](#step-2--retrieve-sources-from-sharepoint)
   - [Step 3 — Retrieve Source Content (HTTP)](#step-3--retrieve-source-content-http)
   - [Step 4 — Change Detection](#step-4--change-detection)
   - [Step 5 — AI Summarization](#step-5--ai-summarization)
   - [Step 6 — Classification Logic](#step-6--classification-logic)
   - [Step 7 — Generate the Markdown Bulletin](#step-7--generate-the-markdown-bulletin)
   - [Step 8 — Teams Notification (Controlled)](#step-8--teams-notification-controlled)
7. [Setup — Phase 3: Build the Weekly Flow](#setup--phase-3-build-the-weekly-flow)
8. [Monitoring & Troubleshooting](#monitoring--troubleshooting)
9. [Adding or Removing Sources](#adding-or-removing-sources)
10. [Governance & Compliance](#governance--compliance)
11. [File Reference](#file-reference)

---

## Overview

This solution eliminates manual bulletin monitoring for a Nova Scotia credit union IT and security team. Two scheduled Power Automate flows — one daily, one weekly — pull content from approved external sources, detect changes using a content hash, summarise new content using the approved AI prompt, classify the output, and file a Markdown report in Microsoft Teams. A Teams channel alert is only sent when the classification is **Attention Required**, keeping notifications meaningful and infrequent.

**What you gain:**

- Consistent daily and weekly bulletins with zero manual effort
- Change-detection that prevents duplicate or stale reports
- Controlled, auditable AI output (same prompt every time)
- A clear escalation path with no AI tool making decisions
- A complete audit trail inside M365 (SharePoint version history + Power Automate run history)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Power Automate (Cloud Flow)                │
│                                                             │
│  Scheduled Trigger (Daily / Weekly)                         │
│       │                                                     │
│       ▼                                                     │
│  Get items → Bulletin Sources (SharePoint List)             │
│       │                                                     │
│       ▼  [Apply to each source]                             │
│  HTTP GET source URL  ──► Member source?                    │
│       │                        │ Yes → Ingest email OR      │
│       │                        │       flag Manual Review   │
│       ▼                        │                            │
│  Generate content hash         │                            │
│       │                        │                            │
│       ▼                        │                            │
│  Hash changed?                 │                            │
│   No  → Log "no change"        │                            │
│   Yes → AI Builder summarize ◄─┘                            │
│              │                                              │
│              ▼                                              │
│  Classify: Attention Required / Awareness Only              │
│              │                                              │
│              ▼                                              │
│  Create Markdown file → SharePoint (Teams > IT > Bulletins) │
│              │                                              │
│              ▼  [Attention Required only]                   │
│  Post Teams message (link to file, one sentence)            │
└─────────────────────────────────────────────────────────────┘
```

Everything stays inside the M365 tenant. No data is sent to third-party services.

---

## Repository Structure

```
.
├── README.md                        # This file — setup & instructions
├── Document.pdf                     # Original specification document
├── config/
│   └── bulletin_sources.json        # Seed data for the Bulletin Sources SharePoint list
├── docs/
│   └── SOP.md                       # Standard Operating Procedure & governance notes
├── flow/
│   └── flow_overview.md             # Detailed Power Automate build reference
├── prompts/
│   └── summarization_prompt.md      # AI Builder system prompt (use verbatim)
└── templates/
    └── bulletin_template.md         # Markdown output template for bulletin files
```

---

## Prerequisites

Before you begin, confirm the following are available in your M365 tenant:

| Requirement | Details |
|-------------|---------|
| **Microsoft 365 licence** | Power Automate Standard or Premium per-user plan (Premium required for HTTP connector) |
| **Power Automate access** | Make sure the account building the flow is a licensed Power Automate user |
| **AI Builder / Copilot** | AI Builder credits or a Copilot Studio licence for the "Create text with GPT" action |
| **SharePoint site** | An existing Teams IT SharePoint site with list-creation permissions |
| **Teams channel** | `Teams > IT` channel where bulletins will be stored and notifications posted |
| **Shared mailbox** | A monitored shared mailbox to receive FS-ISAC (member-only) digest emails |
| **Flow Owner account** | A service or user account that will own both flows and has permissions to SharePoint, Teams, and AI Builder |

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
2. Inside `Bulletins`, create two sub-folders:
   - `Daily Monitoring`
   - `Weekly Monitoring`

---

## Setup — Phase 2: Build the Daily Flow

Open [Power Automate](https://make.powerautomate.com) and follow these steps.

### Step 1 — Create the Scheduled Flow

1. Select **Create → Scheduled cloud flow**.
2. Set the following:
   - **Name:** `Daily – External IT & Cyber Bulletin Monitor`
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

2. Add action: **Initialize variable**
   - **Name:** `Classification`
   - **Type:** String
   - **Value:** *(leave blank)*

3. Add action: **Apply to each**
   - **Select an output from previous steps:** `value` (output of Get items)

All remaining steps in this section are built **inside** the Apply to each loop.

### Step 3 — Retrieve Source Content (HTTP)

Inside the loop, add a **Condition** to check the source type:

- **Condition:** `@{items('Apply_to_each')?['Source_x0020_Type']}` **is not equal to** `Member`

**If Yes (Public or Signal source):**

1. Add action: **HTTP**
   - **Method:** `GET`
   - **URI:** `@{items('Apply_to_each')?['Source_x0020_URL']}`
   - **Headers:** *(none required for public sources)*
   - Rename this action to `HTTP_GetSource` for easier referencing later.

**If No (Member source — e.g., FS-ISAC):**

Choose one of the following approaches:

- **Email ingestion approach:**
  1. Add action: **Get emails (V3)** (Office 365 Outlook)
     - **Mailbox Address:** your monitored shared mailbox address
     - **Folder:** Inbox
     - **Filter:** `From` contains `fsisac.com`
     - **Top:** `5`
  2. Use the email body as the content for summarization.

- **Manual flag approach:**
  1. Add action: **Compose** with value: `"Manual review required — FS-ISAC content must be reviewed directly."`
  2. Add action: **Create item** (SharePoint) to write a `Manual Review` status to the Run Log list.
  3. Add a **Terminate** action (set to `Succeeded`) to skip the rest of the loop iteration for this source.

### Step 4 — Change Detection

After the HTTP action (inside the "Yes" branch of the Source Type condition):

1. Add action: **Compose** — encode the response body for comparison:
   ```
   @{base64(body('HTTP_GetSource'))}
   ```
   Rename this action to `Compose_Hash`.

2. Add action: **Condition** — check whether content has changed:
   - **Left side:** `@{outputs('Compose_Hash')}`
   - **Operator:** `is not equal to`
   - **Right side:** `@{items('Apply_to_each')?['Last_x0020_Reviewed_x0020_Hash']}`

**If No (content unchanged):**

1. Add action: **Create item** (SharePoint — Bulletin Run Log list)
   - Source Name: `@{items('Apply_to_each')?['Source_x0020_Name']}`
   - Run Date: `@{utcNow()}`
   - Status: `No Change`
   - Notes: `No new or updated content identified.`
2. End this iteration (no further actions needed; the loop moves to the next source).

**If Yes (content changed):** Continue to Step 5.

### Step 5 — AI Summarization

Inside the "Yes (changed)" branch:

1. Add action: **Create text with GPT** (AI Builder)  
   *(Search for "AI Builder" in the action picker; the action may also appear as "Create text with GPT using a prompt".)*

2. **System / Instruction Prompt** — paste this **verbatim** (do not modify):

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

3. **User Content:** `@{body('HTTP_GetSource')}` (truncate to ~3,000 characters if the source page is very large to stay within AI Builder token limits).

4. Rename this action to `AIBuilder_Summarize`.

> The full prompt is also stored in `prompts/summarization_prompt.md`. Do not modify the prompt between runs — consistency is required for audit defensibility.

### Step 6 — Classification Logic

After the AI Builder action:

1. Add action: **Set variable**
   - **Name:** `Classification`
   - **Value:** `Awareness Only`

2. Add action: **Condition** — check the summary for high-priority phrases:
   - Build four OR conditions checking if `@{outputs('AIBuilder_Summarize')?['text']}` **contains** any of:
     - `Immediate action required`
     - `Active exploitation`
     - `Service disruption`
     - `Critical vulnerability`

3. **If Yes:** Add action: **Set variable** — set `Classification` to `Attention Required`.

4. **If No:** No action needed (the variable is already `Awareness Only`).

### Step 7 — Generate the Markdown Bulletin

After the classification step:

1. Add action: **Create file** (SharePoint)
   - **Site Address:** Teams IT SharePoint site
   - **Folder Path:** `/Bulletins/Daily Monitoring`
   - **File Name:**
     ```
     @{formatDateTime(utcNow(), 'yyyy-MM-dd')} – @{items('Apply_to_each')?['Source_x0020_Name']} – Automated IT & Cyber Watch.md
     ```
   - **File Content:** Copy the template below (based on `templates/bulletin_template.md`), replacing placeholders with dynamic expressions:

     ```markdown
     # Daily IT & Cyber Watch

     **Date:** @{formatDateTime(utcNow(), 'yyyy-MM-dd')}
     **Source:** @{items('Apply_to_each')?['Source_x0020_Name']}
     **URL:** @{items('Apply_to_each')?['Source_x0020_URL']}
     **Classification:** @{variables('Classification')}

     ---

     @{outputs('AIBuilder_Summarize')?['text']}

     ---

     > This summary was generated automatically.
     > Official guidance remains with the issuing authority.
     > Final incident classification, escalation, and regulatory reporting decisions remain the responsibility of the Credit Union.
     ```

2. After the file is created, update the hash in the SharePoint list:
   - Add action: **Update item** (SharePoint — Bulletin Sources list)
     - **Id:** `@{items('Apply_to_each')?['ID']}`
     - **Last Reviewed Hash:** `@{outputs('Compose_Hash')}`

3. Add action: **Create item** (SharePoint — Bulletin Run Log list)
   - Source Name: `@{items('Apply_to_each')?['Source_x0020_Name']}`
   - Run Date: `@{utcNow()}`
   - Status: `Changed`
   - Notes: `@{variables('Classification')}`

### Step 8 — Teams Notification (Controlled)

After the file creation, add a **Condition**:

- **Condition:** `@{variables('Classification')}` **is equal to** `Attention Required`

**If Yes:**

1. Add action: **Post message in a chat or channel** (Microsoft Teams)
   - **Post as:** Flow bot
   - **Post in:** Channel
   - **Team:** IT
   - **Channel:** General (or a dedicated `#bulletins` channel)
   - **Message:**
     ```
     ⚠️ Attention Required – new bulletin posted for @{items('Apply_to_each')?['Source_x0020_Name']}.
     Review the full bulletin: [paste SharePoint file link or use the file URL from the Create file action output]
     ```
   - **Do not** paste the AI summary into the Teams message body.

**If No (Awareness Only):** No Teams message is sent. The bulletin file is available in SharePoint for daily review.

---

## Setup — Phase 3: Build the Weekly Flow

1. In Power Automate, open the daily flow.
2. Select the **⋯ menu → Save As**.
3. Name the copy: `Weekly – External IT & Cyber Bulletin Monitor`
4. Make the following changes:
   - **Recurrence:** Change frequency to `Week`, interval `1`.
   - **Folder path (Step 7):** Change to `/Bulletins/Weekly Monitoring`
   - **AI Builder prompt:** Append the following sentence to the instruction prompt:
     ```
     Focus on trends and recurring risks rather than individual daily alerts.
     ```
5. Save and turn on the flow.

Everything else (source list, change detection, classification, notification logic) is identical.

---

## Monitoring & Troubleshooting

### Normal Operations

- Check **Power Automate Run History** daily for the first two weeks after go-live.
- Go to **My Flows → Daily – External IT & Cyber Bulletin Monitor → Run history**.
- Each run should complete with status `Succeeded`. Review any `Failed` runs immediately.

### Setting Up Failure Alerts

1. Open the flow.
2. Select **⋯ → Settings**.
3. Enable **Send run failure email** and confirm the recipient address.

### Common Failure Causes

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| HTTP action fails | Source URL changed or site is down | Update the URL in the Bulletin Sources list; verify site availability |
| AI Builder action fails | Quota exceeded or input too large | Check AI Builder credit balance; truncate input content |
| SharePoint action fails | Permissions or column name mismatch | Verify the flow owner account has Contribute access to the SharePoint list and document library |
| Flow times out | Too many sources running serially | Split large source lists into separate flows |

### Manual Fallback

If the flow fails and cannot be restored the same day:

1. Visit each enabled source URL directly.
2. Copy relevant content into a new file using `templates/bulletin_template.md`.
3. Save the file manually to `Teams > IT > Bulletins > Daily Monitoring`.
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

> **Automated summaries are used for awareness only.**  
> Final incident classification, escalation, and regulatory reporting decisions remain the responsibility of the Credit Union.

This statement appears in every generated bulletin. It must not be removed from the template.

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
| `config/bulletin_sources.json` | Seed data for the Bulletin Sources SharePoint list |
| `docs/SOP.md` | Standard Operating Procedure, roles, escalation, audit |
| `flow/flow_overview.md` | Condensed Power Automate build reference |
| `prompts/summarization_prompt.md` | AI Builder system prompt and classification keyword table |
| `templates/bulletin_template.md` | Markdown template used in the "Create file" action |

---

## Flows Summary

| Flow Name | Schedule | Output Folder |
|-----------|----------|---------------|
| Daily – External IT & Cyber Bulletin Monitor | Daily @ 07:00 AST | `Teams > IT > Bulletins > Daily Monitoring` |
| Weekly – External IT & Cyber Bulletin Monitor | Weekly | `Teams > IT > Bulletins > Weekly Monitoring` |
