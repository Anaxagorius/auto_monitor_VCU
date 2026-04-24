# Copilot Summarization Prompt

Use this prompt verbatim in the **manual Copilot review step** (Phase 2B). Open the source content in Word or OneDrive, select the text, and paste this prompt into Copilot. Do not modify the prompt between uses — consistency is required for audit defensibility.

> **Note:** This prompt is for manual use with Microsoft 365 Copilot (inside Word, OneDrive, or Teams). It is not configured inside Power Automate. The automated flow creates intake files only; summarization is a human-in-the-loop step.

---

## Copilot Prompt (Use Verbatim)

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

---

## Weekly Review Variation

When performing the weekly Copilot review, append the following sentence to the prompt above:

```
Focus on trends and recurring risks rather than individual daily alerts.
```

---

## User Content (Dynamic)

Paste or copy the extracted page text before submitting the prompt to Copilot.

```
{{Extracted Page Text}}
```

---

## Classification Keywords

After reviewing the Copilot summary, apply classification based on the presence of any of the following phrases:

| Phrase | Classification |
|--------|----------------|
| "Immediate action required" | Attention Required |
| "Active exploitation" | Attention Required |
| "Service disruption" | Attention Required |
| "Critical vulnerability" | Attention Required |
| *(none of the above)* | Awareness Only |
