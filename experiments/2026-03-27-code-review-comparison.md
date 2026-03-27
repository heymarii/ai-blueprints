# Experiment: Claude vs GPT-4o on Code Review Quality

**Date:** 2026-03-27  
**Hypothesis:** Claude will catch more security issues; GPT-4o will give more actionable fix suggestions

---

## Setup

- **Models:** Claude Sonnet 3.5, GPT-4o
- **Prompt:** Used the [Code Review Prompt v1.3](../prompts/code-gen/code-reviewer.md) for both
- **Test cases:** 5 Python scripts with intentionally planted bugs (3 security, 2 logic)

## Results

| Issue Type | Claude Found | GPT-4o Found |
|------------|-------------|--------------|
| SQL Injection | ✅ | ✅ |
| Missing auth check | ✅ | ❌ |
| XSS vulnerability | ✅ | ✅ |
| Off-by-one error | ✅ | ✅ |
| Race condition | ✅ | ❌ |

**Fix quality (subjective 1-5):**
- Claude: 4.2/5 — detailed, specific, sometimes verbose
- GPT-4o: 3.8/5 — cleaner code examples, but missed edge cases

## Conclusion

Hypothesis partially confirmed. Claude caught all 5 issues; GPT-4o missed 2 of the more subtle ones (auth check, race condition). GPT-4o's suggested fixes had cleaner formatting.

**Recommendation:** Use Claude for security-sensitive reviews. Either is fine for style/readability.

---

*This is one data point — results may vary significantly by codebase and domain.*
