# Standard Operating Procedure
## Automated IT & Cyber Bulletin Monitoring

**System:** Automated Bulletin Collection & Manual Copilot Summarization (Power Automate + M365 Copilot)  
**Owner:** IT / Security Team  
**Audience:** IT Staff, Compliance, Risk  
**Review Frequency:** Annual or after any significant change

---

## 1. Purpose

This SOP governs the automated collection of external IT and cybersecurity bulletin sources and the manual Copilot-assisted summarization process. It defines roles, responsibilities, review cadence, and escalation procedures. The Power Automate flow creates intake files using standard connectors only; summarization and classification are performed manually by IT staff using Microsoft 365 Copilot.

---

## 2. Governance Statement

> "Copilot-assisted summaries are used for awareness only.  
> Final incident classification, escalation, and regulatory reporting decisions remain the responsibility of the Credit Union."

This statement must be included in all bulletins. IT staff must review every Copilot output before applying any classification or taking action.

---

## 3. Roles & Responsibilities

| Role | Responsibility |
|------|----------------|
| Power Automate Flow Owner | Maintain flow health, monitor run history, update source list |
| IT Security Lead / IT Ops | Perform daily Copilot review of intake files; apply classification; create final bulletins; decide escalation |
| Compliance Officer | Annual review of this SOP; sign-off on regulatory relevance |
| Branch Staff | No direct role; escalation comes from IT only |

---

## 4. Daily Workflow

**Phase 1 — Automated (Power Automate)**

1. Flow runs automatically each morning (Atlantic time).
2. Each enabled source in the **Bulletin Sources** SharePoint list is retrieved.
3. For Public and Signal sources: a dated intake file is created in `Teams > IT > Bulletins > Intake` containing the source name, URL, and date.
4. For Member sources (e.g., FS-ISAC): email ingestion from the shared mailbox, or a Manual Review flag is written to the run log.
5. A log entry is written to the **Bulletin Run Log** SharePoint list for each source.

**Phase 2 — Manual (IT Staff + Copilot)**

1. IT Ops opens each intake file from `Teams > IT > Bulletins > Intake`.
2. Visits the source URL listed in the file.
3. Copies relevant content and uses **Microsoft 365 Copilot** (in Word or OneDrive) with the approved prompt (see `prompts/summarization_prompt.md`) to generate a summary.
4. Reviews the Copilot output — does not accept without reading.
5. Applies classification:
   - **Attention Required** — immediate action, active exploitation, service disruption, or critical vulnerability
   - **Awareness Only** — all other cases
6. Fills in `templates/bulletin_template.md` and saves the completed bulletin to `Teams > IT > Bulletins > Daily Monitoring`.
7. If classification = **Attention Required**, posts a Teams message with a link to the bulletin file.
8. IT Security Lead reviews Attention Required bulletins and determines if escalation is warranted.

---

## 5. Weekly Workflow

1. A separate weekly flow runs once per week.
2. Uses the same sources and Phase 1 process as the daily flow (intake files created in `Teams > IT > Bulletins > Intake`).
3. IT staff Copilot review focuses on trends and recurring risks rather than point-in-time alerts (append this to the approved prompt).
4. Output saved to `Teams > IT > Bulletins > Weekly Monitoring`.

---

## 6. Source Management

- All monitored sources are defined in the **Bulletin Sources** SharePoint list.
- Adding or removing sources requires approval from the IT Security Lead.
- Member-only sources (e.g., FS-ISAC) are **not scraped**. They are ingested via summary emails delivered to a monitored shared mailbox, or flagged as "Manual review required."
- The `config/bulletin_sources.json` file in this repository is the authoritative seed for the SharePoint list.

---

## 7. Escalation Path

```
Bulletin classified as Attention Required
  → IT Security Lead reviews bulletin
    → Confirmed threat / action item?
      Yes → Notify affected teams; open incident ticket if warranted
      No  → Log as reviewed; no further action
```

Regulatory reporting decisions (e.g., OSFI incident reporting) remain the sole responsibility of the designated Compliance Officer.

---

## 8. Audit Trail

- Every flow run is logged in Power Automate run history (retained per M365 policy).
- Each intake file and bulletin file is timestamped and stored in SharePoint (version history enabled).
- The Bulletin Run Log list records per-source per-run status (Intake Created, Manual Review, Error).
- The IT staff member's name is recorded in each completed bulletin file.
- No bulletin or intake file is deleted; they are archived in place.

---

## 9. What to Do if the Flow Fails

1. Check Power Automate run history for the error step.
2. Common causes: SharePoint permissions changed, folder path not found, Bulletin Sources list modified.
3. Manual fallback: visit source URLs directly, copy content, use Copilot with the approved prompt, and fill in a bulletin using `templates/bulletin_template.md`.
4. Notify the Flow Owner to restore automated operation within one business day.

---

## 10. Annual Review Checklist

- [ ] Verify all source URLs are still active and relevant
- [ ] Review Copilot prompt for continued accuracy and tone
- [ ] Confirm SharePoint retention settings are appropriate
- [ ] Confirm Teams notification recipients are current
- [ ] Re-read governance statement and confirm it is still displayed in all bulletins
- [ ] Confirm IT staff are using the approved prompt verbatim
- [ ] Update this SOP if any process changes were made during the year
