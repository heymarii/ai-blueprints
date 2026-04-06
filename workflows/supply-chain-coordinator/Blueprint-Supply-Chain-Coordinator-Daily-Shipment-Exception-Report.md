# Blueprint: Supply Chain Coordinator — Daily Shipment & Exception Report

> **Role:** Supply Chain Coordinator / Logistics Analyst
> **Task automated:** Daily shipment tracking, exception flagging, carrier performance scoring, and stakeholder delay notifications
> **Time saved:** ~12–15 hours/week
> **Output:** A prioritized exception dashboard with delay root-cause classification, carrier scorecards, and auto-drafted escalation emails

---

## The Problem

Supply chain coordinators start every morning the same way: logging into 3–5 systems (TMS, carrier portals, ERP, email) to manually check the status of dozens or hundreds of shipments. They copy tracking numbers into carrier websites, cross-reference ETAs against customer commit dates, flag anything running late, then spend the rest of the morning writing emails to carriers, warehouses, and account managers explaining what's delayed and why.

**Typical daily breakdown (manual):**

| Task | Time Spent |
|------|-----------|
| Logging into carrier portals & pulling tracking updates | 45 min |
| Cross-referencing ETAs against commit dates in ERP | 30 min |
| Classifying delay types and root causes | 30 min |
| Writing exception emails to carriers | 45 min |
| Writing delay notifications to account managers/customers | 30 min |
| Updating shipment status spreadsheet | 20 min |
| Pulling carrier on-time performance data for weekly reviews | 30 min |
| **Total** | **~3.5 hrs/day** |

This is high-stakes but low-complexity work — it follows clear rules, involves structured data, and produces standardized outputs. A perfect candidate for automation.

---

## The Automated Workflow

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    TRIGGER (6:00 AM)                     │
│             Scheduled daily / on-demand run              │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│              STEP 1: DATA COLLECTION                    │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │   TMS    │  │ Carrier  │  │   ERP    │  │ Email  │ │
│  │   API    │  │   APIs   │  │  System  │  │ Inbox  │ │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───┬────┘ │
│       │              │             │             │       │
│       └──────────────┴─────────────┴─────────────┘      │
│                         │                               │
└─────────────────────────┼───────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────┐
│           STEP 2: NORMALIZATION & MATCHING               │
│                                                          │
│  • Unify shipment IDs across systems                     │
│  • Match current carrier ETA → customer commit date      │
│  • Compute variance (days early/late)                    │
│  • Flag shipments where ETA > commit date                │
└─────────────────────────┬───────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────┐
│         STEP 3: EXCEPTION CLASSIFICATION (AI)            │
│                                                          │
│  For each flagged shipment, classify root cause:         │
│                                                          │
│  🔴 CRITICAL  — 3+ days late, high-value, or customer   │
│                  escalation risk                         │
│  🟡 WARNING   — 1-2 days late or at risk of missing     │
│                  commit date                             │
│  🟢 ON TRACK  — ETA within commit window                │
│                                                          │
│  Root cause categories:                                  │
│  • Carrier delay (transit)    • Weather/force majeure    │
│  • Customs/border hold        • Warehouse processing     │
│  • Carrier pickup missed      • Documentation error      │
│  • Capacity/equipment issue   • Unknown (needs research) │
└─────────────────────────┬───────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────┐
│        STEP 4: CARRIER PERFORMANCE SCORING               │
│                                                          │
│  Rolling 30-day scorecard per carrier:                   │
│  • On-time delivery %                                    │
│  • Average transit variance (days)                       │
│  • Exception frequency by type                           │
│  • Damage/claim rate                                     │
│  • Responsiveness score (avg reply time on exceptions)   │
└─────────────────────────┬───────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────┐
│          STEP 5: REPORT GENERATION & ACTIONS             │
│                                                          │
│  📊 Exception Dashboard (HTML/PDF)                       │
│  📧 Auto-drafted escalation emails (carrier + internal)  │
│  📋 Updated shipment tracker (spreadsheet/ERP)           │
│  📈 Carrier scorecard update                             │
└─────────────────────────────────────────────────────────┘
```

---

## Implementation

### Option A: Low-Code (Recommended for most teams)

**Tools:** Zapier/Make.com + Google Sheets + Claude API + Gmail

#### Step-by-step setup:

1. **Data ingestion (Zapier/Make.com)**
   - Connect carrier APIs (FedEx, UPS, USPS, or your 3PL's API) via HTTP modules
   - Pull open shipment list from your TMS or ERP via API/webhook
   - Schedule to run daily at 6:00 AM

2. **Normalization (Google Sheets + Apps Script)**
   - Incoming data lands in a master Google Sheet
   - Apps Script normalizes carrier status codes into a unified format
   - Computes ETA vs. commit date variance

3. **AI classification (Claude API call)**
   - Send each exception to Claude with the prompt below
   - Claude returns severity, root cause, and recommended action

4. **Output generation (Claude API + Gmail)**
   - Claude generates the dashboard summary and drafts emails
   - Zapier sends emails via Gmail (held in draft for human review)

#### Core Prompt — Exception Classification

```
You are a supply chain exception analyst. Given shipment data, classify the exception and recommend an action.

SHIPMENT DATA:
- Shipment ID: {{shipment_id}}
- Carrier: {{carrier_name}}
- Origin: {{origin}} → Destination: {{destination}}
- Ship date: {{ship_date}}
- Original ETA: {{original_eta}}
- Current carrier ETA: {{current_eta}}
- Customer commit date: {{commit_date}}
- Last tracking event: {{last_event}}
- Last event timestamp: {{last_event_time}}
- Shipment value: ${{value}}
- Customer tier: {{customer_tier}}

CLASSIFICATION RULES:
- CRITICAL (🔴): Shipment is 3+ days past commit date, OR value > $50,000 and any delay, OR customer tier is "Enterprise" and delay > 1 day
- WARNING (🟡): Shipment is 1-2 days past commit date, OR current ETA is after commit date but shipment hasn't arrived
- ON TRACK (🟢): Current ETA is on or before commit date

Respond in this exact JSON format:
{
  "severity": "CRITICAL|WARNING|ON_TRACK",
  "delay_days": <number>,
  "root_cause": "<category from: carrier_transit_delay, weather_force_majeure, customs_hold, warehouse_processing, missed_pickup, documentation_error, capacity_equipment, unknown>",
  "root_cause_confidence": "<high|medium|low>",
  "recommended_action": "<specific next step>",
  "escalation_needed": <true|false>,
  "carrier_email_draft": "<2-3 sentence email to carrier requesting update/resolution>",
  "internal_notification": "<1-2 sentence summary for account manager>"
}
```

#### Core Prompt — Daily Dashboard Summary

```
You are a supply chain reporting analyst. Generate a concise executive summary of today's shipment exceptions.

TODAY'S DATA:
- Total active shipments: {{total_shipments}}
- On track: {{on_track_count}} ({{on_track_pct}}%)
- Warning: {{warning_count}} ({{warning_pct}}%)
- Critical: {{critical_count}} ({{critical_pct}}%)

CRITICAL SHIPMENTS:
{{critical_shipments_json}}

WARNING SHIPMENTS:
{{warning_shipments_json}}

CARRIER PERFORMANCE (30-day rolling):
{{carrier_scorecard_json}}

Generate a report with these sections:
1. **Executive Summary** — 3-sentence overview of today's shipment health
2. **Critical Exceptions** — Table with shipment ID, carrier, delay days, root cause, and action needed
3. **Warnings to Watch** — Brief list of at-risk shipments
4. **Carrier Scorecard** — Performance ranking with on-time %, trend arrow (↑↓→)
5. **Recommended Actions** — Prioritized list of what the coordinator should do first today

Keep the tone professional and action-oriented. Use tables for structured data. Flag any patterns (e.g., "FedEx Ground has had 4 delays from the Chicago hub this week — consider routing alternatives").
```

### Option B: Python Script (For teams with developer support)

```python
"""
Daily Shipment Exception Report Generator
Run via cron at 6:00 AM or trigger on-demand
"""

import json
import os
from datetime import datetime, timedelta
from anthropic import Anthropic

# --- Configuration ---
CARRIERS = {
    "fedex": {"api_url": "https://apis.fedex.com/track/v1/trackingnumbers", "api_key": os.getenv("FEDEX_API_KEY")},
    "ups": {"api_url": "https://onlinetools.ups.com/api/track/v1/details", "api_key": os.getenv("UPS_API_KEY")},
}
CLAUDE_MODEL = "claude-sonnet-4-6"
client = Anthropic()

def fetch_open_shipments():
    """Pull open shipments from TMS/ERP. Replace with your actual data source."""
    # Example: query your database or API
    # return requests.get(f"{TMS_API}/shipments?status=in_transit").json()
    return [
        {
            "shipment_id": "SHP-2026-04891",
            "carrier": "fedex",
            "tracking_number": "794644790132",
            "origin": "Chicago, IL",
            "destination": "Dallas, TX",
            "ship_date": "2026-04-03",
            "original_eta": "2026-04-06",
            "commit_date": "2026-04-06",
            "value": 12500,
            "customer_tier": "Standard",
        },
        {
            "shipment_id": "SHP-2026-04903",
            "carrier": "ups",
            "tracking_number": "1Z999AA10123456784",
            "origin": "Los Angeles, CA",
            "destination": "New York, NY",
            "ship_date": "2026-04-01",
            "original_eta": "2026-04-05",
            "commit_date": "2026-04-04",
            "value": 78000,
            "customer_tier": "Enterprise",
        },
    ]

def get_carrier_tracking(carrier, tracking_number):
    """Fetch latest tracking status from carrier API. Implement per carrier."""
    # Replace with actual API calls
    # response = requests.post(CARRIERS[carrier]["api_url"], ...)
    return {
        "current_eta": "2026-04-07",
        "last_event": "In transit - Delayed due to weather",
        "last_event_time": "2026-04-05T14:30:00Z",
        "status": "in_transit",
    }

def classify_exception(shipment, tracking):
    """Use Claude to classify the exception severity and root cause."""
    prompt = f"""You are a supply chain exception analyst. Classify this shipment exception.

SHIPMENT:
- ID: {shipment['shipment_id']}
- Carrier: {shipment['carrier']}
- Route: {shipment['origin']} → {shipment['destination']}
- Ship date: {shipment['ship_date']}
- Original ETA: {shipment['original_eta']}
- Current ETA: {tracking['current_eta']}
- Commit date: {shipment['commit_date']}
- Last event: {tracking['last_event']}
- Value: ${shipment['value']:,}
- Customer tier: {shipment['customer_tier']}

Respond in JSON with: severity, delay_days, root_cause, recommended_action,
carrier_email_draft, internal_notification."""

    response = client.messages.create(
        model=CLAUDE_MODEL,
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}],
    )
    return json.loads(response.content[0].text)

def generate_dashboard(exceptions, carrier_scores):
    """Generate the full daily dashboard report."""
    prompt = f"""Generate a daily shipment exception dashboard report.

EXCEPTIONS: {json.dumps(exceptions, indent=2)}
CARRIER SCORES: {json.dumps(carrier_scores, indent=2)}
DATE: {datetime.now().strftime('%B %d, %Y')}

Include: Executive summary, critical exceptions table, warnings list,
carrier scorecard, and prioritized action items. Format as clean markdown."""

    response = client.messages.create(
        model=CLAUDE_MODEL,
        max_tokens=4096,
        messages=[{"role": "user", "content": prompt}],
    )
    return response.content[0].text

def main():
    shipments = fetch_open_shipments()
    exceptions = []

    for shipment in shipments:
        tracking = get_carrier_tracking(shipment["carrier"], shipment["tracking_number"])
        commit = datetime.strptime(shipment["commit_date"], "%Y-%m-%d")
        current_eta = datetime.strptime(tracking["current_eta"], "%Y-%m-%d")

        if current_eta > commit:
            classification = classify_exception(shipment, tracking)
            exceptions.append({**shipment, **tracking, **classification})

    # Generate carrier scorecards (simplified — pull from historical data in production)
    carrier_scores = {
        "fedex": {"on_time_pct": 91.2, "avg_delay_days": 0.8, "trend": "stable"},
        "ups": {"on_time_pct": 87.5, "avg_delay_days": 1.3, "trend": "declining"},
    }

    dashboard = generate_dashboard(exceptions, carrier_scores)

    # Save report
    filename = f"shipment_report_{datetime.now().strftime('%Y-%m-%d')}.md"
    with open(filename, "w") as f:
        f.write(dashboard)

    # Send emails (integrate with your email system)
    for exc in exceptions:
        if exc.get("severity") == "CRITICAL":
            print(f"⚠️  CRITICAL: {exc['shipment_id']} — {exc['recommended_action']}")
            # send_email(to=carrier_contact, body=exc["carrier_email_draft"])
            # send_email(to=account_manager, body=exc["internal_notification"])

    print(f"\n✅ Report generated: {filename}")
    print(f"   {len(exceptions)} exceptions flagged out of {len(shipments)} active shipments")

if __name__ == "__main__":
    main()
```

---

## Example Output

### Daily Shipment Exception Dashboard — April 6, 2026

#### Executive Summary

Of 47 active shipments, 38 (80.9%) are on track, 6 (12.8%) are in warning status, and 3 (6.4%) are critical. FedEx Ground continues to show delays out of the Chicago hub — this is the third consecutive day with weather-related exceptions on that lane. UPS's on-time rate dropped below 88% for the rolling 30-day window, driven by capacity issues on West Coast → East Coast routes.

#### Critical Exceptions

| Shipment ID | Carrier | Route | Delay | Root Cause | Commit Date | Value | Action |
|-------------|---------|-------|-------|------------|-------------|-------|--------|
| SHP-2026-04903 | UPS | LA → NYC | 3 days | Capacity/equipment | Apr 4 | $78,000 | Escalate to UPS regional manager; notify Enterprise account team; evaluate air freight recovery |
| SHP-2026-04887 | FedEx Ground | Chicago → Miami | 2 days | Weather | Apr 5 | $34,200 | Monitor hourly; pre-draft customer delay notice for trigger at EOD |
| SHP-2026-04912 | Regional LTL | Atlanta → Seattle | 4 days | Missed pickup | Apr 3 | $15,800 | File carrier claim; rebook via FedEx Freight for next-day pickup |

#### Carrier Scorecard (30-Day Rolling)

| Carrier | On-Time % | Trend | Avg Delay | Exception Rate | Action |
|---------|-----------|-------|-----------|----------------|--------|
| FedEx Express | 96.1% | → Stable | 0.3 days | 3.9% | No action needed |
| FedEx Ground | 91.2% | ↓ Declining | 0.8 days | 8.8% | Review Chicago hub routing |
| UPS Ground | 87.5% | ↓ Declining | 1.3 days | 12.5% | Schedule QBR; request corrective action plan |
| Regional LTL | 82.0% | ↓ Declining | 1.9 days | 18.0% | Evaluate alternative carriers for Southeast lanes |

#### Prioritized Actions for Today

1. **Call UPS regional manager** about SHP-2026-04903 ($78K Enterprise shipment, 3 days late) — request expedited recovery and root cause
2. **Send pre-emptive delay notice** to Acme Corp (Enterprise) regarding potential late delivery on SHP-2026-04903
3. **Rebook SHP-2026-04912** via FedEx Freight — the Regional LTL missed pickup is unrecoverable on the original routing
4. **Monitor FedEx Ground Chicago lane** — if SHP-2026-04887 doesn't clear weather hold by 2 PM, trigger customer notification
5. **Flag UPS performance decline** for next week's carrier review meeting — on-time below 88% triggers contract review clause

---

## Why This Should Be Implemented

**Quantifiable impact:**

- **Time saved:** 12–15 hours/week (2.5–3 hrs/day × 5 days)
- **Faster exception response:** From ~3 hours (manual morning review) to ~15 minutes (AI report delivered at 6 AM, coordinator reviews and acts by 6:15 AM)
- **Reduced missed SLAs:** Proactive flagging catches at-risk shipments 1–2 days earlier than reactive tracking
- **Carrier accountability:** Automated scorecards create data-driven leverage for rate negotiations and QBRs
- **Customer satisfaction:** Enterprise accounts get proactive delay notifications instead of reactive apologies

**ROI calculation:**
- Coordinator salary (avg): ~$55,000/year → ~$26.44/hr
- Hours saved: 12 hrs/week × 50 weeks = 600 hrs/year
- Labor savings: 600 × $26.44 = **$15,864/year per coordinator**
- Claude API cost: ~$50–100/month = $600–1,200/year
- **Net savings: ~$14,600–15,200/year per coordinator**

For a team of 5 coordinators, that's **$73,000–76,000/year** — and the real value is in the SLA improvements and customer retention that faster exception handling delivers.

---

## Getting Started

1. **Week 1:** Set up carrier API connections and pull shipment data into a central spreadsheet
2. **Week 2:** Implement the Claude classification prompt and test against 1 week of historical exceptions
3. **Week 3:** Add email drafting and scorecard generation; run in parallel with manual process
4. **Week 4:** Go live — coordinator reviews AI-generated report each morning and takes action

**Prerequisites:**
- Carrier API access (FedEx, UPS, etc.) or 3PL portal credentials
- TMS or ERP with API/export capability
- Claude API key (Anthropic)
- Zapier/Make.com account OR Python environment for Option B

---

*Blueprint created: April 6, 2026 | Model: Claude Opus 4.6*
