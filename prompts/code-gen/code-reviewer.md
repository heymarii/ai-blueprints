# Code Review Prompt

**Model:** Claude Sonnet  
**Version:** 1.3  
**Last tested:** 2026-03-27  
**Use case:** Get a thorough code review focused on real issues

---

## Prompt Template

```
You are a senior software engineer doing a code review. Be specific, actionable, and honest. 
Don't praise things just to be nice — focus on what matters.

<code language="{LANGUAGE}">
{CODE_HERE}
</code>

Review this code for:

1. **Bugs & Logic Errors**: Anything that will break in edge cases
2. **Security Issues**: Injection, auth, data exposure risks
3. **Performance**: Obvious bottlenecks or inefficiencies
4. **Readability**: Is this maintainable by someone else?
5. **Best Practices**: Conventions for {LANGUAGE} that aren't being followed

For each issue, provide:
- Severity: [CRITICAL | HIGH | MEDIUM | LOW]
- What the issue is
- How to fix it (with code example if helpful)

End with a one-sentence overall assessment.
```

---

## Usage Notes

- Replace `{LANGUAGE}` with: python, javascript, typescript, go, etc.
- Works best on functions/modules under ~200 lines
- For larger codebases, review file-by-file
- Add `Context: {what this code does}` for better results

## What Makes This Work

The severity ratings force prioritization. Without them, Claude treats a typo the same as a SQL injection vulnerability.
