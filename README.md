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
- **[Insurance Claims Adjuster: Daily Claims Triage Report](workflows/insurance-claims-adjuster/Blueprint-Insurance-Claims-Adjuster-Daily-Claims-Triage-Report.md)** — AI-powered claims triage, severity classification, fraud detection, and missing documentation tracking for claims adjusters (~12–15h/week saved)
- **[Procurement Specialist: Vendor Quote Comparison Report](workflows/procurement-specialist/Blueprint-Procurement-Specialist-Vendor-Quote-Comparison-Report.md)** — Automated vendor quote extraction, normalization, weighted comparison scoring, and PO draft generation for procurement teams (~15–20h/week saved)
- **[Restaurant Manager: Daily Inventory & Waste Report](workflows/restaurant-manager/Blueprint-Restaurant-Manager-Daily-Inventory-Waste-Report.md)** — Automated inventory tracking, waste detection, and reorder suggestions for restaurant managers (~10h/week saved)
- **[Sales Rep: Daily Lead Qualification & Follow-Up Report](workflows/sales-rep/Blueprint-Sales-Rep-Daily-Lead-Qualification-Follow-Up-Report.md)** — Automated lead scoring, ICP matching, personalized outreach drafts, and daily prioritized action plans for SDRs and account executives (~10–12h/week saved)
- **[Content Creator: Weekly Performance Report & Calendar](workflows/content-creator/Blueprint-Content-Creator-Weekly-Performance-Report-and-Calendar.md)** — Automated cross-platform analytics collection, performance analysis, trend detection, and AI-generated content calendar for social media managers and creators (~10h/week saved)
- **[Teacher: Weekly Student Progress Report](workflows/teacher/Blueprint-Teacher-Weekly-Student-Progress-Report.md)** — Automated student data analysis, individualized progress narratives, priority flagging, and parent email drafts for K-12 teachers (~10-12h/week saved)
- **[Product Manager: Weekly Stakeholder Status Report](workflows/product-manager/Blueprint-Product-Manager-Weekly-Stakeholder-Status-Report.md)** — Automated sprint data aggregation, ticket-to-deployment correlation, Slack signal extraction, and AI-generated stakeholder narratives for PMs (~8h/week saved)
- **[Supply Chain Coordinator: Daily Shipment & Exception Report](workflows/supply-chain-coordinator/Blueprint-Supply-Chain-Coordinator-Daily-Shipment-Exception-Report.md)** — Automated shipment tracking, exception classification, carrier performance scoring, and escalation email drafting for logistics teams (~12–15h/week saved)
- **[Customer Support Manager: Daily Ticket Triage & Escalation Report](workflows/customer-support-manager/Blueprint-Customer-Support-Manager-Daily-Ticket-Triage-Escalation-Report.md)** — AI-powered ticket classification, SLA monitoring, pattern detection, agent performance analysis, and churn risk scoring for support managers (~10h/week saved)
- **[HR Recruiter: Automated Candidate Screening & Interview Pipeline](workflows/hr-recruiter/Blueprint-HR-Recruiter-Automated-Candidate-Screening-Pipeline.md)** — AI-powered resume screening, candidate scoring, personalized outreach generation, intelligent interview scheduling, and pipeline health monitoring for recruiters (~12h/week saved)
- **[Executive Assistant: Daily Meeting Prep & Action Item Tracker](workflows/executive-assistant/Blueprint-Executive-Assistant-Daily-Meeting-Prep-Action-Item-Tracker.md)** — Automated meeting context enrichment, daily briefing packet generation, post-meeting action item extraction, recap email drafting, and follow-up scheduling for EAs (~12–15h/week saved)
- **[Office Administrator: Expense Report Processing & Approval Tracker](workflows/office-administrator/Blueprint-Office-Administrator-Expense-Report-Processing-Approval-Tracker.md)** — Automated receipt intake via OCR, AI-powered categorization, rules-based approval routing, weekly dashboard generation, and credit card reconciliation for office admins (~10–12h/week saved)

### Experiments
- **[Code Review Comparison](experiments/2026-03-27-code-review-comparison.md)** — Model comparison for code review quality

## Quick Start

Browse the `prompts/` folder for drop-in templates, `agents/` for full architecture starters, or `workflows/` for end-to-end automation blueprints organized by role.
