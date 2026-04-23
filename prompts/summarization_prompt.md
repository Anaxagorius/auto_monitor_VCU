# AI Summarization Prompt

Use this prompt verbatim in the **"Create text with GPT"** (or Copilot text generation) action inside Power Automate. Do not modify the system/instruction prompt between runs — consistency is required for audit defensibility.

---

## System / Instruction Prompt

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

## User Content (Dynamic)

Paste or map the extracted page text here. Truncate to the AI Builder token limit if needed.

```
{{Extracted Page Text}}
```

---

## Classification Keywords

After summarization, apply a condition step that scans the output for the following phrases to determine classification:

| Phrase | Classification |
|--------|----------------|
| "Immediate action required" | Attention Required |
| "Active exploitation" | Attention Required |
| "Service disruption" | Attention Required |
| "Critical vulnerability" | Attention Required |
| *(none of the above)* | Awareness Only |
