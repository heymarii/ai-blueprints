# 🤖 AI Blueprints

> A library of reusable AI workflow blueprints, prompt templates, and agent architecture patterns. Actively maintained and updated as I build and test new approaches.

## What's Here

```
ai-blueprints/
├── prompts/           ← Tested, versioned prompt templates by use case
├── agents/            ← Agent architecture patterns and starter configs
├── workflows/         ← Multi-step AI workflow blueprints
├── evaluations/       ← How to test and score AI outputs
└── experiments/       ← Notes from AI experiments and model comparisons
```

## Philosophy

Each blueprint here has been **tested in production or personal projects**. I version these because prompts drift — what worked last month may not work after a model update.

## Available Workflows

### Prompts
- **[Code Reviewer](prompts/code-gen/code-reviewer.md)** — Prompt template for automated code review
- **[Universal Summarizer](prompts/summarization/universal-summarizer.md)** — Flexible summarization prompt for any content type

### Agents
- **[Research Agent](agents/research-agent.md)** — Agent architecture for autonomous research tasks

### Workflows
- **[Daily Brief](workflows/daily-brief.md)** — Personalized morning briefing combining news, calendar, and tasks
- **[Restaurant Manager: Daily Inventory & Waste Report](workflows/restaurant-manager/Blueprint-Restaurant-Manager-Daily-Inventory-Waste-Report.md)** — Automated inventory tracking, waste detection, and reorder suggestions for restaurant managers (~10h/week saved)

### Experiments
- **[Code Review Comparison](experiments/2026-03-27-code-review-comparison.md)** — Model comparison for code review quality

## Quick Start

Browse the `prompts/` folder for drop-in templates, `agents/` for full architecture starters, or `workflows/` for end-to-end automation blueprints organized by role.

---

*Updated: March 27, 2026 | Model: Claude Sonnet (primary), GPT-4o (secondary)*
