# Universal Summarizer Prompt

**Model:** Claude Sonnet / GPT-4o  
**Version:** 2.1  
**Last tested:** 2026-03-27  
**Use case:** Summarize any document, article, or transcript

---

## Prompt Template

```
You are a precise summarizer. Your job is to extract the most important information from the provided content.

<content>
{CONTENT_HERE}
</content>

Please provide:

1. **TL;DR** (1-2 sentences): The single most important takeaway
2. **Key Points** (3-5 bullets): The most important facts, decisions, or insights
3. **Action Items** (if any): Any explicit next steps or decisions required
4. **Context** (1 sentence): Who this is for and why it matters

Format your response in clean Markdown. Be concise — if something isn't important, leave it out.
```

---

## Usage Notes

- Works well on: articles, meeting transcripts, documentation, emails
- Replace `{CONTENT_HERE}` with your actual content
- For very long documents (>10k tokens), chunk first
- Add `Audience: {role}` before the content block to tailor vocabulary

## Version History

- v2.1 (2026-03-27): Added "Context" section, improved action item detection
- v2.0 (2026-02): Restructured output format, added numbered sections
- v1.0 (2026-01): Initial version

## Example Output

> **TL;DR:** New GitHub API feature allows file creation without a local git client, enabling fully serverless commit workflows.
> 
> **Key Points:**
> - PUT endpoint accepts base64-encoded file content
> - SHA required for updates (not creates)
> - Works with fine-grained PATs scoped per-repo
