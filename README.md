# Automated Bulletin Monitoring & Summarization

**Platform:** Microsoft Power Automate (Cloud Flow)  
**Output:** Markdown files in Microsoft Teams (SharePoint)  
**Audience:** IT / Security / Compliance  
**Human Role:** Oversight, approval, and escalation only

---

## Overview

This solution automatically monitors external IT and cybersecurity bulletin sources, detects changes, summarizes content using Copilot / AI Builder, and saves structured Markdown reports to Microsoft Teams. Everything runs inside the M365 tenant.

```
Scheduled Trigger
  → Retrieve Source Content
  → Detect Changes
  → Summarize with Copilot
  → Generate Markdown Bulletin
  → Save to Teams > IT > Bulletins
  → (Optional) Notify Channel if Action Required
```

---

## Repository Structure

```
.
├── README.md                        # This file
├── Document.pdf                     # Original specification
├── config/
│   └── bulletin_sources.json        # Controlled source list (import into SharePoint)
├── docs/
│   └── SOP.md                       # Standard Operating Procedure & governance notes
├── flow/
│   └── flow_overview.md             # Step-by-step Power Automate build guide
├── prompts/
│   └── summarization_prompt.md      # AI summarization system prompt
└── templates/
    └── bulletin_template.md         # Markdown output template for bulletins
```

---

## Quick-Start

### Prerequisites

- Microsoft 365 licence with Power Automate (Standard or Premium)
- SharePoint site for the IT team
- Copilot Studio / AI Builder access (for the GPT text generation action)
- A monitored shared mailbox for FS-ISAC email ingestion (member-only source)

### Setup Steps

1. **Create the SharePoint list** – Use `config/bulletin_sources.json` as the column schema and seed data.  
2. **Import the flow** – Follow `flow/flow_overview.md` to build both the daily and weekly flows.  
3. **Configure the prompt** – Copy the content from `prompts/summarization_prompt.md` into the AI Builder action.  
4. **Set the output path** – Point the "Create file" action at `Teams > IT > Bulletins > Daily Monitoring`.  
5. **Review governance** – Read and distribute `docs/SOP.md` before go-live.

---

## Governance Statement

> Automated summaries are used for awareness only.  
> Final incident classification, escalation, and regulatory reporting decisions remain the responsibility of the Credit Union.

---

## Flows

| Flow Name | Schedule | Output Folder |
|-----------|----------|---------------|
| Daily – External IT & Cyber Bulletin Monitor | Daily (early Atlantic business hours) | Teams > IT > Bulletins > Daily Monitoring |
| Weekly – External IT & Cyber Bulletin Monitor | Weekly | Teams > IT > Bulletins > Weekly Monitoring |
