# Power Automate Flow Build Guide

This document walks through every step to build the **Daily – External IT & Cyber Bulletin Monitor** flow in Power Automate. The weekly flow is a duplicate with minor changes noted at the end.

---

## Step 1 – Create the Flow

1. Open [Power Automate](https://make.powerautomate.com).
2. Select **Create → Scheduled cloud flow**.
3. Name the flow: `Daily – External IT & Cyber Bulletin Monitor`
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
| Last Reviewed Hash | Single line of text | Populated by flow; leave blank initially |

3. Populate with the entries from `config/bulletin_sources.json`.

---

## Step 3 – Retrieve Content

Inside the flow, after the trigger:

1. Add action: **Get items** (SharePoint)
   - Site: your Teams IT SharePoint site
   - List: Bulletin Sources
   - Filter: `Enabled eq 1`

2. Add action: **Apply to each** (loop over returned items)

3. Inside the loop — for **Public** and **Signal** sources:
   - Add action: **HTTP**
     - Method: `GET`
     - URI: `@{items('Apply_to_each')?['Source URL']}`

4. For **Member** sources (FS-ISAC):
   - **Do not scrape.**
   - Add action: **Get emails (V3)** from the monitored shared mailbox, filtered by sender domain.
   - OR add a **Compose** action outputting `"Manual review required"` and skip summarization.

---

## Step 4 – Change Detection

Inside the loop, after HTTP retrieval:

1. Add action: **Compose** — generate a hash of the response body:
   ```
   @{base64(body('HTTP'))}
   ```
   *(A full SHA hash requires a custom connector or Azure Function; base64 is sufficient for most change detection needs.)*

2. Add action: **Condition**
   - Condition: `Compose output` **is not equal to** `@{items('Apply_to_each')?['Last Reviewed Hash']}`

3. **If No (unchanged):**
   - Add action: **Create item** or **Update item** (SharePoint) to log `"No new or updated content identified"` to a run log list.
   - End the current iteration (do nothing further).

4. **If Yes (changed):**
   - Continue to Step 5.
   - After summarization completes, add action: **Update item** (SharePoint) to write the new hash back to `Last Reviewed Hash`.

---

## Step 5 – Summarization

Inside the "If Yes" branch:

1. Add action: **Create text with GPT** (AI Builder) or **Copilot text generation**.
2. Set the **System / Instruction Prompt** to the exact text in `prompts/summarization_prompt.md`.
3. Set the **User Content** to the HTTP response body (truncated to token limit if needed).

---

## Step 6 – Decision Logic (Action vs. Awareness)

After summarization:

1. Add action: **Condition**
   - Check if the AI output **contains** any of:
     - `Immediate action required`
     - `Active exploitation`
     - `Service disruption`
     - `Critical vulnerability`

2. **If Yes:** Set a variable `Classification` = `Attention Required`
3. **If No:** Set a variable `Classification` = `Awareness Only`

---

## Step 7 – Generate the Markdown File

1. Add action: **Create file** (SharePoint)
   - Site: Teams IT SharePoint site
   - Folder path: `Teams > IT > Bulletins > Daily Monitoring`
   - File name:
     ```
     @{formatDateTime(utcNow(), 'yyyy-MM-dd')} – Automated IT & Cyber Watch.md
     ```
   - File content: Use the template from `templates/bulletin_template.md`, substituting dynamic values:
     - `{{YYYY-MM-DD}}` → `@{formatDateTime(utcNow(), 'yyyy-MM-dd')}`
     - `{{Source Name}}` → `@{items('Apply_to_each')?['Source Name']}`
     - `{{Source URL}}` → `@{items('Apply_to_each')?['Source URL']}`
     - `{{AI Summary Text}}` → AI Builder output
     - `{{Awareness Only | Attention Required}}` → `Classification` variable

---

## Step 8 – Teams Notification (Controlled)

After file creation, add another **Condition**:

- If `Classification` **equals** `Attention Required`:
  1. Add action: **Post message in a chat or channel** (Microsoft Teams)
  2. Post to: `Teams > IT` channel
  3. Message:
     ```
     ⚠️ Attention Required – new bulletin posted for @{items('Apply_to_each')?['Source Name']}.
     Review: [link to SharePoint file]
     ```
  4. Do **not** paste the summary into the chat message.

- If `Classification` **equals** `Awareness Only`:
  - No notification. File is available in SharePoint for daily review.

---

## Step 9 – Weekly Flow

1. In Power Automate, open the daily flow and select **Save As** → name it `Weekly – External IT & Cyber Bulletin Monitor`.
2. Change recurrence to **Weekly**.
3. Change the output folder path to: `Teams > IT > Bulletins > Weekly Monitoring`
4. In the AI Builder prompt, append this sentence to the instruction:
   ```
   Focus on trends and recurring risks rather than individual daily alerts.
   ```
5. Everything else remains identical.

---

## Flow Health & Monitoring

- Check Power Automate **Run History** daily for the first two weeks after go-live.
- Set up a **Flow failure notification** under Flow settings → Send run failure email.
- Review SharePoint run log list weekly to confirm sources with no changes are still active.
