# Blueprint: Automated Expense Report Processing & Approval Tracker

**Role:** Office Administrator  
**Task:** Expense Report Collection, Categorization, Approval Routing & Reconciliation  
**Time Saved:** ~10-12 hours/week  
**Difficulty to Implement:** Low-Medium  
**Tools Required:** Email (Gmail/Outlook), Google Sheets or Excel, OCR API (Google Vision / AWS Textract), Zapier or Make, Slack or Teams  

---

## The Problem

Office administrators are the backbone of financial operations in small-to-midsize companies. Every week, they spend hours on a painfully manual expense workflow:

1. **Collecting receipts** — Chasing employees via email, Slack, or in person to submit receipts before deadlines. This alone can eat 2-3 hours/week.
2. **Manual data entry** — Typing receipt details (vendor, amount, date, category) into spreadsheets or expense systems. Error-prone and tedious.
3. **Categorization** — Deciding whether each expense is Travel, Meals, Office Supplies, Software, etc., and tagging it to the right cost center or project.
4. **Approval routing** — Figuring out who needs to approve what based on amount thresholds and department, then sending emails and following up on stalled approvals.
5. **Reconciliation** — Matching approved expenses against credit card statements and flagging discrepancies.

This workflow is repetitive, rule-based, and ripe for automation.

---

## The Automated Workflow

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    EXPENSE AUTOMATION PIPELINE               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────┐    ┌──────────┐    ┌───────────┐              │
│  │ Employee  │───>│  Email/  │───>│  OCR      │              │
│  │ Submits   │    │  Form    │    │  Extract  │              │
│  │ Receipt   │    │  Intake  │    │  Data     │              │
│  └──────────┘    └──────────┘    └─────┬─────┘              │
│                                        │                     │
│                                        ▼                     │
│                                 ┌──────────────┐             │
│                                 │ AI Categorize │             │
│                                 │ & Validate    │             │
│                                 └──────┬───────┘             │
│                                        │                     │
│                              ┌─────────┴─────────┐          │
│                              ▼                   ▼           │
│                     ┌──────────────┐    ┌──────────────┐     │
│                     │  < $500      │    │  >= $500     │     │
│                     │  Auto-approve│    │  Route to    │     │
│                     │  + Log       │    │  Manager     │     │
│                     └──────┬───────┘    └──────┬───────┘     │
│                            │                   │             │
│                            ▼                   ▼             │
│                     ┌────────────────────────────────┐       │
│                     │   Master Expense Spreadsheet    │       │
│                     │   + Weekly Summary Dashboard    │       │
│                     └────────────────────────────────┘       │
│                                    │                         │
│                                    ▼                         │
│                     ┌────────────────────────────────┐       │
│                     │  Credit Card Reconciliation    │       │
│                     │  (Flag mismatches automatically)│       │
│                     └────────────────────────────────┘       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Implementation

### Step 1: Receipt Intake (Replace manual collection)

**Setup a dedicated intake channel.** Employees forward receipts to a single email address (e.g., `expenses@company.com`) or submit via a simple Google Form.

**Zapier/Make Trigger:**
- **Trigger:** New email arrives at `expenses@company.com` with attachment
- **Action:** Save attachment to a Google Drive folder (`/Expenses/Inbox/`)
- **Action:** Log submission in tracking sheet with timestamp, sender, and status "Processing"

**Google Form Alternative:**
- Fields: Employee Name, Date, Amount, Category (dropdown), Receipt Upload, Notes
- On submit: auto-populates the master spreadsheet and saves receipt to Drive

**Prompt for automated receipt reminder (runs Monday 9 AM):**

```
You are an office assistant. Draft a friendly Slack message reminding 
the team to submit any outstanding receipts from last week. 

Include:
- A deadline (Wednesday EOD)
- A link to the submission form
- A note that late submissions delay reimbursement

Keep the tone warm but professional. Use 3-4 sentences max.
```

**Example Output:**
> Hey team! Quick reminder to submit any expense receipts from last week by **Wednesday EOD**. You can drop them here: [Expense Form Link]. Late submissions may delay your reimbursement, so please get them in on time. Thanks!

---

### Step 2: OCR Data Extraction (Replace manual data entry)

Use an OCR service to automatically read receipt images and extract structured data.

**Python Script — Receipt OCR Extraction:**

```python
import json
from google.cloud import vision
from datetime import datetime

def extract_receipt_data(image_path: str) -> dict:
    """Extract vendor, amount, date, and line items from a receipt image."""
    client = vision.ImageAnnotatorClient()
    
    with open(image_path, "rb") as f:
        content = f.read()
    
    image = vision.Image(content=content)
    response = client.text_detection(image=image)
    full_text = response.text_annotations[0].description if response.text_annotations else ""
    
    # Send extracted text to Claude for structured parsing
    parsed = parse_receipt_with_ai(full_text)
    return parsed


def parse_receipt_with_ai(raw_text: str) -> dict:
    """
    Prompt for Claude/GPT to structure raw OCR text into expense fields.
    In production, call the API. Here we show the prompt.
    """
    prompt = f"""
    Extract the following fields from this receipt text. Return valid JSON only.
    
    Fields:
    - vendor_name: string
    - date: YYYY-MM-DD
    - total_amount: float
    - tax_amount: float (0 if not found)
    - payment_method: string (credit card, cash, etc.)
    - suggested_category: one of [Travel, Meals & Entertainment, Office Supplies, 
      Software & Subscriptions, Professional Services, Utilities, Other]
    - confidence: float 0-1 (how confident you are in the extraction)
    
    Receipt text:
    ---
    {raw_text}
    ---
    
    Return JSON only, no explanation.
    """
    # In production: response = anthropic.messages.create(model="claude-sonnet-4-6", ...)
    # Return parsed JSON
    return json.loads('{"vendor_name": "...", "total_amount": 0, ...}')  # placeholder


# Example usage
if __name__ == "__main__":
    result = extract_receipt_data("receipt_photo.jpg")
    print(json.dumps(result, indent=2))
```

**Example Extracted Output:**
```json
{
  "vendor_name": "Office Depot",
  "date": "2026-04-07",
  "total_amount": 234.56,
  "tax_amount": 18.76,
  "payment_method": "Visa ending 4521",
  "suggested_category": "Office Supplies",
  "confidence": 0.94
}
```

---

### Step 3: AI Categorization & Validation

**Prompt for expense categorization (runs per receipt):**

```
You are a financial categorization assistant for a mid-size company.

Given the following expense details, determine:
1. The correct expense category from this list:
   [Travel, Meals & Entertainment, Office Supplies, Software & Subscriptions, 
    Professional Services, Utilities, Marketing, Training & Education, Other]
2. The appropriate cost center based on the submitter's department
3. Whether this expense requires any flags:
   - DUPLICATE: Similar amount/vendor within 7 days
   - OVER_LIMIT: Exceeds department monthly budget threshold
   - MISSING_INFO: Key fields are empty or low-confidence
   - POLICY_VIOLATION: Violates company expense policy (e.g., alcohol, personal items)

Expense:
- Submitter: {employee_name}, Department: {department}
- Vendor: {vendor_name}
- Amount: ${amount}
- Date: {date}
- Description: {notes}
- Recent expenses from this employee: {recent_history}

Return JSON with: category, cost_center, flags[], flag_reasons[], approved_auto (bool)
```

---

### Step 4: Approval Routing

**Rules Engine (configurable, no code needed):**

| Condition | Action |
|-----------|--------|
| Amount < $100, no flags | Auto-approve, log to sheet |
| Amount $100-$500, no flags | Auto-approve, notify manager via Slack |
| Amount >= $500 | Route to manager for approval via Slack/email |
| Any flags present | Route to admin for manual review |
| POLICY_VIOLATION flag | Route to finance director |

**Slack Approval Message Template:**

```
Expense Approval Request

Employee: Sarah Chen (Marketing)
Vendor: Delta Airlines
Amount: $847.00
Category: Travel
Date: 2026-04-05
Notes: Flight to Chicago for Q2 client meeting

[Approve]  [Reject]  [Request More Info]

This request requires your approval (amount >= $500).
Please respond within 48 hours.
```

**Escalation automation:** If no response in 48 hours, send a follow-up. If no response in 72 hours, escalate to the next manager up.

---

### Step 5: Weekly Dashboard & Reconciliation

**The automated weekly summary consolidates everything into a single report.**

**Prompt for weekly expense summary (runs Friday 4 PM):**

```
You are a financial reporting assistant. Generate a weekly expense summary 
from the following data. Format it as a clean, professional report.

Data: {weekly_expense_json}

Include these sections:
1. EXECUTIVE SUMMARY - Total spend, number of reports, comparison to last week
2. BY CATEGORY - Breakdown of spend per category with % of total
3. BY DEPARTMENT - Which departments spent what
4. FLAGGED ITEMS - Any expenses that need attention (duplicates, policy issues, 
   pending approvals)
5. RECONCILIATION STATUS - How many expenses matched credit card statements, 
   how many have discrepancies
6. ACTION ITEMS - What the admin needs to do next (follow up on X, 
   investigate Y, process reimbursement for Z)

Keep it concise. Use tables where helpful. Highlight anything unusual.
```

**Example Weekly Summary Output:**

---

### Weekly Expense Report — Apr 3-9, 2026

**Executive Summary:** 47 expenses processed totaling $12,847.33 (down 8% from last week). 41 auto-approved, 4 manager-approved, 2 pending review.

| Category | Amount | % of Total | vs. Last Week |
|----------|--------|------------|---------------|
| Travel | $4,230.00 | 32.9% | +15% |
| Meals & Entertainment | $2,115.50 | 16.5% | -3% |
| Office Supplies | $1,890.23 | 14.7% | -22% |
| Software & Subscriptions | $3,450.00 | 26.8% | +5% |
| Professional Services | $1,161.60 | 9.0% | +2% |

**Flagged Items:**
- DUPLICATE: Two Uber charges from Jake M. on 4/7 ($34.50 each) — likely duplicate
- OVER_LIMIT: Marketing dept at 92% of monthly budget with 3 weeks remaining
- PENDING: 2 approvals awaiting response from VP Engineering (sent 4/7)

**Reconciliation:** 45/47 matched to credit card statements. 2 discrepancies flagged for manual review (amounts differ by >$5).

**Action Items:**
1. Follow up with Jake M. on potential duplicate Uber charge
2. Alert Marketing director about budget threshold
3. Escalate VP Engineering approvals (48hr+ pending)
4. Investigate 2 reconciliation mismatches

---

## What This Replaces

| Manual Task | Time Before | Time After | How |
|-------------|-------------|------------|-----|
| Chasing receipt submissions | 2-3 hrs/wk | 0 (automated reminders + form) | Zapier + Slack bot |
| Data entry from receipts | 3-4 hrs/wk | 15 min (review OCR results) | Google Vision + AI parsing |
| Categorization | 1-2 hrs/wk | 5 min (review AI suggestions) | Claude prompt |
| Approval routing & follow-up | 2-3 hrs/wk | 10 min (handle exceptions) | Rules engine + Slack |
| Weekly reporting | 1-2 hrs/wk | 0 (auto-generated) | Scheduled prompt |
| Reconciliation | 1-2 hrs/wk | 15 min (review flags only) | Script matching |
| **TOTAL** | **10-16 hrs/wk** | **~45 min/wk** | |

---

## Why Implement This

**For the Office Administrator:** You go from being a data-entry bottleneck to a strategic operator. Your time shifts from typing numbers into spreadsheets to reviewing dashboards, handling exceptions, and improving processes. You become the person who *runs* the system, not the person *stuck in* the system.

**For the Business:** Faster reimbursements mean happier employees. Automated policy enforcement means fewer violations slipping through. Real-time budget tracking means no more end-of-month surprises. The ROI on this automation pays for itself within the first month.

**For Compliance:** Every expense has a digital audit trail — who submitted it, when, how it was categorized, who approved it, and whether it matched the credit card statement. No more lost receipts or missing approvals.

---

## Getting Started (30-Minute Setup)

1. **Create the intake form** (10 min) — Set up a Google Form with the fields listed above. Share the link with your team.
2. **Connect Zapier/Make** (10 min) — Create a zap: Form submission → Add row to Google Sheet → Send Slack confirmation to submitter.
3. **Set up the approval Slack channel** (5 min) — Create `#expense-approvals` and configure the routing rules.
4. **Schedule the weekly report prompt** (5 min) — Use a scheduled task to run the summary prompt every Friday at 4 PM.
5. **Add OCR later** — Start with the form-based intake. Once the workflow is running smoothly, layer in OCR for receipt photo processing.

---

*Built by heymarii | AI Workflow Blueprints | April 2026*
