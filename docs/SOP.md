# Standard Operating Procedure
## Automated IT & Cyber Bulletin Monitoring

**System:** Automated Bulletin Monitoring & Summarization (Power Automate)  
**Owner:** IT / Security Team  
**Audience:** IT Staff, Compliance, Risk  
**Review Frequency:** Annual or after any significant change

---

## 1. Purpose

This SOP governs the automated monitoring of external IT and cybersecurity bulletin sources. It defines roles, responsibilities, review cadence, and escalation procedures.

---

## 2. Governance Statement

> "Automated summaries are used for awareness only.  
> Final incident classification, escalation, and regulatory reporting decisions remain the responsibility of the Credit Union."

This statement must be included in all generated bulletins and reviewed by IT staff before any action is taken.

---

## 3. Roles & Responsibilities

| Role | Responsibility |
|------|----------------|
| Power Automate Flow Owner | Maintain flow health, monitor run history, update source list |
| IT Security Lead | Review Attention Required bulletins; decide escalation |
| Compliance Officer | Annual review of this SOP; sign-off on regulatory relevance |
| Branch Staff | No direct role; escalation comes from IT only |

---

## 4. Daily Workflow

1. Flow runs automatically each morning (Atlantic time).
2. Each enabled source in the **Bulletin Sources** SharePoint list is retrieved via HTTP GET.
3. A content hash is compared to the stored `LastReviewedHash` value.
   - **No change detected:** Log entry written; no bulletin generated.
   - **Change detected:** Summarization proceeds; hash updated after completion.
4. AI summary is generated using the approved prompt (see `prompts/summarization_prompt.md`).
5. Markdown bulletin is saved to `Teams > IT > Bulletins > Daily Monitoring`.
6. If classification = **Attention Required**, a single Teams message is posted with a link to the bulletin file.
7. IT Security Lead reviews and determines if escalation is warranted.

---

## 5. Weekly Workflow

1. A separate weekly flow runs once per week.
2. Uses the same sources and process as the daily flow.
3. Prompt focus: trends and recurring risks rather than point-in-time alerts.
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
- Each bulletin file is timestamped and stored in SharePoint (version history enabled).
- The `LastReviewedHash` column in the Bulletin Sources list provides change-detection evidence.
- No bulletin is deleted; they are archived in place.

---

## 9. What to Do if the Flow Fails

1. Check Power Automate run history for the error step.
2. Common causes: source URL changed, HTTP timeout, AI Builder quota exceeded.
3. Manual fallback: visit source URLs directly and document findings in a bulletin using `templates/bulletin_template.md`.
4. Notify the Flow Owner to restore automated operation within one business day.

---

## 10. Annual Review Checklist

- [ ] Verify all source URLs are still active and relevant
- [ ] Review AI prompt for continued accuracy and tone
- [ ] Confirm SharePoint retention settings are appropriate
- [ ] Confirm Teams notification recipients are current
- [ ] Re-read governance statement and confirm it is still displayed in all bulletins
- [ ] Update this SOP if any process changes were made during the year
