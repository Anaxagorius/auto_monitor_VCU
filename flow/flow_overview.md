# Power Automate Flow Build Guide

This document walks through every step to build the **Daily – External IT & Cyber Bulletin Collector** flow in Power Automate. The weekly flow is a duplicate with minor changes noted at the end.

> **License note:** This design uses **standard connectors only**. The HTTP connector and AI Builder ("Create text with GPT") are premium and are not used. Summarization is performed manually by IT staff using Copilot inside Word or OneDrive (see Phase 2B in the README).

---

## Step 1 – Create the Flow

1. Open [Power Automate](https://make.powerautomate.com).
2. Select **Create → Scheduled cloud flow**.
3. Name the flow: `Daily – External IT & Cyber Bulletin Collector`
4. Set recurrence:
   - **Frequency:** Day
   - **Interval:** 1
   - **Start time:** Early Atlantic business hours (e.g., 07:00 AST / 11:00 UTC)
5. Save the flow.

---

## Step 2 – Define the Source List (SharePoint)

1. In your Teams IT site, create a SharePoint list named **Bulletin Sources**.
2. Add the following columns:

| Column Name | Type | Notes |
|-------------|------|-------|
| Source Name | Single line of text | Display name |
| Source URL | Single line of text | Full HTTPS URL |
| Source Type | Choice | Options: Public, Member, Signal |
| Monitoring Frequency | Choice | Options: Daily, Weekly |
| Enabled | Yes/No | Default: Yes |
| Last Reviewed Hash | Single line of text | Reserved for future use; leave blank |

3. Populate with the entries from `config/bulletin_sources.json`.

---

## Step 3 – Create Intake Files

Inside the flow, after the trigger:

1. Add action: **Get items** (SharePoint)
   - Site: your Teams IT SharePoint site
   - List: Bulletin Sources
   - Filter: `Enabled eq 1`

2. Add action: **Apply to each** (loop over returned items)

3. Inside the loop — add a **Condition** to check source type:
   - **Condition:** `Source Type` **is not equal to** `Member`

4. **If Yes (Public or Signal source):**
   - Add action: **Create file** (SharePoint)
     - Site: Teams IT SharePoint site
     - Folder: `/Bulletins/Intake`
     - File name: `@{formatDateTime(utcNow(), 'yyyy-MM-dd')} – @{items('Apply_to_each')?['Source Name']} – Raw.txt`
     - File content:
       ```
       Source: [Source Name]
       URL: [Source URL]
       Date: [Today's date]

       ACTION REQUIRED: Open the URL above, copy relevant content,
       then use Copilot to summarise using the approved prompt.
       Save the final bulletin to Bulletins/Daily Monitoring.
       ```
   - Add action: **Create item** (SharePoint — Bulletin Run Log)
     - Status: `Intake Created`

5. **If No (Member source — e.g., FS-ISAC):**
   - **Do not scrape.**
   - Option A: Add action: **Get emails (V3)** from the monitored shared mailbox, filtered by sender domain. Save email body as intake file content.
   - Option B: Add action: **Create file** with content `"Manual review required"` and log `Manual Review` status to the Run Log list.

---

## Step 4 – Manual Copilot Review (IT Staff — Phase 2B)

After the flow runs, IT staff open each intake file and:

1. Visit the source URL listed in the file.
2. Copy relevant content into Word or OneDrive.
3. Ask Copilot using the **full approved prompt verbatim** (from `prompts/summarization_prompt.md`):
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
4. Review the Copilot output — do not accept without reading.
5. Apply classification:
   - **Attention Required:** immediate action, active exploitation, service disruption, or critical vulnerability
   - **Awareness Only:** all other cases
6. Paste into `templates/bulletin_template.md` and save to `Bulletins/Daily Monitoring`.
7. If **Attention Required**, post a Teams message with a link to the bulletin file.

---

## Step 5 – Weekly Flow

1. In Power Automate, open the daily flow and select **Save As** → name it `Weekly – External IT & Cyber Bulletin Collector`.
2. Change recurrence to **Weekly**.
3. In the intake file content, update the folder reference to: `Bulletins/Intake` (same folder; the file name prefix distinguishes daily from weekly if needed).
4. For the manual Copilot review, append to the prompt:
   ```
   Focus on trends and recurring risks rather than individual daily alerts.
   ```
5. Save weekly bulletins to `Teams > IT > Bulletins > Weekly Monitoring`.

---

## Flow Health & Monitoring

- Check Power Automate **Run History** daily for the first two weeks after go-live.
- Set up a **Flow failure notification** under Flow settings → Send run failure email.
- Review SharePoint run log list weekly to confirm all sources are generating intake files.
