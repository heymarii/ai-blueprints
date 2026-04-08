# Blueprint: Executive Assistant — Daily Meeting Prep & Action Item Tracker

> **Role:** Executive Assistant / Administrative Professional
> **Task:** Automate daily meeting preparation, briefing document generation, action item extraction, and follow-up communication drafting
> **Time Saved:** ~12–15 hours/week
> **Difficulty:** Medium
> **Tools:** Google Calendar API (or Outlook), Gmail/Slack API, Google Docs/Notion, AI (Claude or GPT), Zapier/Make

---

## The Problem

Executive assistants are the operational backbone of leadership teams. They spend a disproportionate amount of time on tasks that follow predictable patterns:

- **Meeting prep (3–4h/day):** Pulling context from past emails, prior meeting notes, attendee bios, and relevant documents for each meeting on the executive's calendar.
- **Action item tracking (1–2h/day):** Listening to or reading meeting transcripts, extracting commitments, assigning owners, and updating trackers.
- **Follow-up drafts (1–2h/day):** Writing post-meeting recap emails, scheduling follow-ups, and nudging overdue items.
- **Calendar Tetris (1h/day):** Coordinating across time zones, juggling reschedules, and prioritizing meeting requests.

Most of this work is pattern-based: the information already exists across tools — it just needs to be collected, synthesized, and formatted. That's exactly what AI excels at.

---

## The Automated Workflow

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DAILY MEETING PREP PIPELINE                      │
│                   Trigger: 6:00 AM daily (cron)                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │  1. CALENDAR  │───▶│  2. CONTEXT      │───▶│  3. BRIEFING     │  │
│  │  SCAN         │    │  ENRICHMENT      │    │  GENERATION      │  │
│  │              │    │                  │    │                  │  │
│  │ Pull today's │    │ For each meeting:│    │ AI-generated     │  │
│  │ meetings,    │    │ - Past emails    │    │ one-pager per    │  │
│  │ attendees,   │    │ - Prior notes    │    │ meeting with:    │  │
│  │ agendas      │    │ - LinkedIn/CRM   │    │ - Context brief  │  │
│  │              │    │ - Open action    │    │ - Attendee notes │  │
│  │              │    │   items          │    │ - Suggested Qs   │  │
│  └──────────────┘    └──────────────────┘    │ - Open items     │  │
│                                               └────────┬─────────┘  │
│                                                        │            │
│  ┌──────────────────────────────────────────────────────▼─────────┐ │
│  │                  4. DAILY BRIEFING PACKET                      │ │
│  │                                                                │ │
│  │  Consolidated doc delivered to exec's inbox by 7:00 AM:       │ │
│  │  - Day-at-a-glance schedule with prep notes                   │ │
│  │  - Priority flags (VIP meetings, overdue follow-ups)          │ │
│  │  - Suggested time blocks for deep work                        │ │
│  └────────────────────────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│              POST-MEETING ACTION ITEM PIPELINE                      │
│            Trigger: Meeting ends (calendar event)                   │
│                                                                     │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │  5. TRANSCRIPT│───▶│  6. AI           │───▶│  7. OUTPUTS      │  │
│  │  CAPTURE      │    │  EXTRACTION      │    │                  │  │
│  │              │    │                  │    │ - Action items   │  │
│  │ Otter.ai /   │    │ Parse transcript │    │   → tracker      │  │
│  │ Fireflies /  │    │ for:             │    │ - Recap email    │  │
│  │ Google Meet  │    │ - Decisions made │    │   → draft        │  │
│  │ transcript   │    │ - Action items   │    │ - Follow-up      │  │
│  │              │    │ - Owners + dates │    │   calendar       │  │
│  │              │    │ - Key takeaways  │    │   events         │  │
│  └──────────────┘    └──────────────────┘    └──────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Implementation

### Step 1: Calendar Scan & Attendee Extraction

**Trigger:** Scheduled daily at 6:00 AM via cron job or Zapier/Make.

```python
"""
Step 1: Fetch today's calendar events and extract attendee information.
Uses Google Calendar API. Swap for Microsoft Graph API if using Outlook.
"""
from datetime import datetime, timedelta
from googleapiclient.discovery import build
from google.oauth2.credentials import Credentials

def get_todays_meetings(credentials_path: str, calendar_id: str = "primary"):
    creds = Credentials.from_authorized_user_file(credentials_path)
    service = build("calendar", "v3", credentials=creds)

    now = datetime.utcnow()
    start_of_day = now.replace(hour=0, minute=0, second=0).isoformat() + "Z"
    end_of_day = now.replace(hour=23, minute=59, second=59).isoformat() + "Z"

    events = service.events().list(
        calendarId=calendar_id,
        timeMin=start_of_day,
        timeMax=end_of_day,
        singleEvents=True,
        orderBy="startTime"
    ).execute().get("items", [])

    meetings = []
    for event in events:
        meetings.append({
            "id": event["id"],
            "title": event.get("summary", "No Title"),
            "start": event["start"].get("dateTime", event["start"].get("date")),
            "end": event["end"].get("dateTime", event["end"].get("date")),
            "attendees": [
                {
                    "email": a["email"],
                    "name": a.get("displayName", ""),
                    "response": a.get("responseStatus", "needsAction")
                }
                for a in event.get("attendees", [])
            ],
            "description": event.get("description", ""),
            "location": event.get("location", ""),
            "meeting_link": event.get("hangoutLink", ""),
            "organizer": event.get("organizer", {}).get("email", "")
        })

    return meetings
```

### Step 2: Context Enrichment

For each meeting, the system gathers relevant historical context:

```python
"""
Step 2: Enrich each meeting with context from email, notes, and CRM.
"""
import anthropic

def enrich_meeting_context(meeting: dict, email_client, notes_client, crm_client=None):
    context = {"meeting": meeting, "prior_emails": [], "prior_notes": [], "open_actions": []}

    # Pull recent emails with attendees (last 30 days)
    for attendee in meeting["attendees"]:
        emails = email_client.search(
            query=f"from:{attendee['email']} OR to:{attendee['email']}",
            max_results=10,
            days_back=30
        )
        context["prior_emails"].extend(emails)

    # Pull prior meeting notes mentioning this topic or attendees
    attendee_names = [a["name"] for a in meeting["attendees"] if a["name"]]
    notes = notes_client.search(
        query=f"{meeting['title']} {' '.join(attendee_names)}",
        max_results=5
    )
    context["prior_notes"] = notes

    # Pull open action items assigned to/from attendees
    for attendee in meeting["attendees"]:
        actions = notes_client.get_open_actions(assignee=attendee["email"])
        context["open_actions"].extend(actions)

    # Optional: CRM context for external meetings
    if crm_client and any(not a["email"].endswith("@yourcompany.com") for a in meeting["attendees"]):
        for attendee in meeting["attendees"]:
            if not attendee["email"].endswith("@yourcompany.com"):
                crm_data = crm_client.lookup_contact(attendee["email"])
                if crm_data:
                    context["crm_context"] = crm_data

    return context


def generate_meeting_brief(context: dict) -> str:
    """Use Claude to generate a concise meeting briefing document."""
    client = anthropic.Anthropic()

    prompt = f"""You are an executive assistant preparing a meeting briefing.

Meeting: {context['meeting']['title']}
Time: {context['meeting']['start']} – {context['meeting']['end']}
Attendees: {', '.join(a['name'] or a['email'] for a in context['meeting']['attendees'])}
Agenda/Description: {context['meeting'].get('description', 'None provided')}

Recent email threads with attendees:
{format_emails(context['prior_emails'])}

Prior meeting notes:
{format_notes(context['prior_notes'])}

Open action items involving attendees:
{format_actions(context['open_actions'])}

Generate a meeting briefing with these sections:
1. **Meeting Overview** (1-2 sentences: what this meeting is about and why it matters)
2. **Attendee Notes** (who's attending, their role, last interaction, anything notable)
3. **Context & Background** (key points from recent emails and prior meetings)
4. **Open Items** (unresolved action items that may come up)
5. **Suggested Talking Points** (3-5 questions or topics the exec should raise)
6. **Prep Checklist** (any documents to review, decisions needed, or materials to bring)

Keep it scannable — bullet points and bold headers. Max 300 words per meeting."""

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1500,
        messages=[{"role": "user", "content": prompt}]
    )

    return response.content[0].text
```

### Step 3: Daily Briefing Packet Assembly

```python
"""
Step 3: Assemble all meeting briefs into a single daily packet.
"""
from datetime import datetime

def assemble_daily_packet(meetings_with_briefs: list, exec_name: str = "Executive") -> str:
    today = datetime.now().strftime("%A, %B %d, %Y")

    packet = f"""# Daily Briefing — {today}
**Prepared for:** {exec_name}
**Total meetings today:** {len(meetings_with_briefs)}
**Generated at:** {datetime.now().strftime("%I:%M %p")}

---

## Day at a Glance

| Time | Meeting | Priority | Attendees |
|------|---------|----------|-----------|
"""

    for m in meetings_with_briefs:
        meeting = m["meeting"]
        priority = classify_priority(meeting)
        start = datetime.fromisoformat(meeting["start"]).strftime("%I:%M %p")
        attendee_count = len(meeting["attendees"])
        packet += f"| {start} | {meeting['title']} | {priority} | {attendee_count} |\n"

    # Flag overdue action items
    overdue = [a for m in meetings_with_briefs for a in m.get("open_actions", []) if a.get("overdue")]
    if overdue:
        packet += f"\n> ⚠️ **{len(overdue)} overdue action item(s)** need attention before today's meetings.\n"

    packet += "\n---\n\n## Meeting Briefings\n\n"

    for i, m in enumerate(meetings_with_briefs, 1):
        meeting = m["meeting"]
        start = datetime.fromisoformat(meeting["start"]).strftime("%I:%M %p")
        end = datetime.fromisoformat(meeting["end"]).strftime("%I:%M %p")
        packet += f"### {i}. {meeting['title']} ({start} – {end})\n\n"
        packet += m["brief"] + "\n\n---\n\n"

    # Suggest deep work blocks
    packet += suggest_deep_work_blocks(meetings_with_briefs)

    return packet


def classify_priority(meeting: dict) -> str:
    """Simple priority classification based on signals."""
    title_lower = meeting["title"].lower()
    attendee_count = len(meeting["attendees"])

    if any(kw in title_lower for kw in ["board", "investor", "ceo", "urgent", "escalation"]):
        return "🔴 HIGH"
    elif attendee_count > 5 or any(kw in title_lower for kw in ["review", "planning", "strategy"]):
        return "🟡 MEDIUM"
    else:
        return "🟢 STANDARD"


def suggest_deep_work_blocks(meetings: list) -> str:
    """Identify gaps between meetings for deep work."""
    # Parse meeting times, find gaps > 45 minutes
    section = "## Suggested Deep Work Blocks\n\n"
    # Implementation would parse start/end times and find gaps
    section += "_Gaps of 45+ minutes between meetings are highlighted as potential focus time._\n"
    return section
```

### Step 4: Post-Meeting Action Item Extraction

```python
"""
Step 4: After each meeting, extract action items from the transcript.
Triggered when a meeting ends (via calendar webhook or Zapier).
"""
import anthropic
import json

def extract_action_items(transcript: str, meeting_title: str, attendees: list) -> dict:
    """Parse a meeting transcript and extract structured action items."""
    client = anthropic.Anthropic()

    attendee_list = ", ".join(a["name"] or a["email"] for a in attendees)

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2000,
        messages=[{"role": "user", "content": f"""Analyze this meeting transcript and extract structured data.

Meeting: {meeting_title}
Attendees: {attendee_list}

Transcript:
{transcript}

Return a JSON object with:
{{
  "summary": "2-3 sentence meeting summary",
  "key_decisions": ["list of decisions made"],
  "action_items": [
    {{
      "task": "description of the action item",
      "owner": "person responsible (must be from attendee list)",
      "due_date": "YYYY-MM-DD or 'TBD' if not specified",
      "priority": "high/medium/low",
      "context": "brief context for why this matters"
    }}
  ],
  "follow_ups": ["topics that need follow-up but aren't action items"],
  "parking_lot": ["ideas or topics deferred to future discussion"]
}}

Rules:
- Only extract EXPLICIT commitments as action items (not implied ones)
- Owner must be someone who was in the meeting
- If a due date was discussed, capture it; otherwise mark TBD
- Be specific about what the action item actually requires"""}]
    )

    return json.loads(response.content[0].text)


def generate_recap_email(meeting_data: dict, meeting_title: str) -> str:
    """Generate a professional recap email from extracted meeting data."""
    client = anthropic.Anthropic()

    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1000,
        messages=[{"role": "user", "content": f"""Write a professional meeting recap email.

Meeting: {meeting_title}
Data: {json.dumps(meeting_data, indent=2)}

Format:
- Subject line
- Brief greeting
- Meeting summary (2-3 sentences)
- Key Decisions (bulleted)
- Action Items (table: Task | Owner | Due Date)
- Next Steps
- Professional sign-off

Tone: Professional, concise, clear. No fluff."""}]
    )

    return response.content[0].text
```

### Step 5: Tracker Update & Follow-Up Scheduling

```python
"""
Step 5: Update the action item tracker and create follow-up calendar events.
"""

def update_action_tracker(action_items: list, tracker_client):
    """Push extracted action items to the tracking system (Notion, Asana, etc.)."""
    for item in action_items:
        tracker_client.create_task(
            title=item["task"],
            assignee=item["owner"],
            due_date=item["due_date"],
            priority=item["priority"],
            notes=item["context"],
            status="open"
        )


def schedule_follow_ups(action_items: list, calendar_service):
    """Create calendar reminders for action item due dates."""
    for item in action_items:
        if item["due_date"] != "TBD":
            # Create a reminder event 1 day before due date
            calendar_service.create_event(
                summary=f"Follow up: {item['task']} (Owner: {item['owner']})",
                date=item["due_date"],
                reminder_minutes=1440  # 24 hours before
            )


def send_weekly_accountability_digest(tracker_client, email_client):
    """Weekly digest of all open action items, sent every Friday."""
    open_items = tracker_client.get_open_items()

    overdue = [i for i in open_items if i["overdue"]]
    due_this_week = [i for i in open_items if i["due_this_week"]]

    digest = f"""# Weekly Action Item Digest

## Overdue ({len(overdue)} items)
{format_items_table(overdue)}

## Due This Week ({len(due_this_week)} items)
{format_items_table(due_this_week)}

## All Open Items ({len(open_items)} total)
{format_items_table(open_items)}
"""

    email_client.send(
        to="executive@company.com",
        subject=f"Weekly Action Items: {len(overdue)} overdue, {len(due_this_week)} due this week",
        body=digest
    )
```

---

## Example Output: Daily Briefing Packet

```
╔══════════════════════════════════════════════════════════════════╗
║              DAILY BRIEFING — Wednesday, April 8, 2026         ║
║              Prepared for: Sarah Chen, VP of Operations        ║
║              Generated at: 6:45 AM                             ║
╠══════════════════════════════════════════════════════════════════╣

  DAY AT A GLANCE
  ────────────────────────────────────────────────────────────────
  9:00 AM   Product Roadmap Review          🟡 MEDIUM   6 attendees
  10:30 AM  1:1 with Direct Report (James)  🟢 STANDARD 2 attendees
  1:00 PM   Board Prep: Q1 Financials       🔴 HIGH     4 attendees
  3:00 PM   Vendor Evaluation: CloudSync    🟡 MEDIUM   5 attendees
  4:30 PM   Team Standup                    🟢 STANDARD 8 attendees

  ⚠️ 2 overdue action items need attention before today's meetings.

  ────────────────────────────────────────────────────────────────
  MEETING 1: Product Roadmap Review (9:00 – 10:15 AM)
  ────────────────────────────────────────────────────────────────

  OVERVIEW
  Quarterly roadmap prioritization session with the product and
  engineering leads. Last quarter's roadmap hit 72% of targets —
  the team will likely push to reduce scope this quarter.

  ATTENDEE NOTES
  • Mike Torres (Head of Product) — Sent a pre-read on Monday
    with proposed Q2 priorities. Has been vocal about reducing
    tech debt allocation.
  • Lisa Park (Engineering Lead) — Flagged in Slack yesterday
    that the auth migration is 2 weeks behind.
  • 4 others from product and engineering

  OPEN ITEMS FROM LAST MEETING
  • [OVERDUE] Mike owes final API partner priority ranking (was
    due March 28)
  • Lisa committed to headcount proposal for Q2 — status unknown

  SUGGESTED TALKING POINTS
  1. What's the realistic completion % we should target for Q2?
  2. Should we formally deprioritize the auth migration or add
     resources?
  3. Have we heard back from the API partner on timeline?

  PREP CHECKLIST
  ☐ Review Mike's pre-read (attached to calendar invite)
  ☐ Pull up Q1 roadmap scorecard for reference
  ☐ Decision needed: Approve or modify Q2 priority stack

  ────────────────────────────────────────────────────────────────
  MEETING 3: Board Prep: Q1 Financials (1:00 – 2:00 PM)  🔴
  ────────────────────────────────────────────────────────────────

  OVERVIEW
  Pre-board dry run for the Q1 financial presentation. The board
  meeting is April 15 — this is the last prep session. CFO wants
  to align on the narrative around the revenue miss in EMEA.

  ATTENDEE NOTES
  • David Kim (CFO) — Shared updated deck v3 yesterday evening.
    Key change: reframed EMEA shortfall as "delayed pipeline"
    vs. "miss." Check if Sarah is comfortable with that framing.
  • Rachel Gomez (IR Lead) — Expects board questions on burn
    rate given recent hires.

  OPEN ITEMS
  • Sarah owes final sign-off on slide 14 (customer case study)
  • David requested headcount forecast from HR — not received

  SUGGESTED TALKING POINTS
  1. Are we comfortable with the "delayed pipeline" framing for
     EMEA, or do we need a more transparent narrative?
  2. What's our answer if the board asks about extending runway?
  3. Has the customer approved their case study for the deck?

  PREP CHECKLIST
  ☐ Review deck v3 (Google Drive link in last email from David)
  ☐ Approve or revise slide 14
  ☐ Prepare 2-sentence answer for burn rate question

  ────────────────────────────────────────────────────────────────
  SUGGESTED DEEP WORK BLOCKS
  ────────────────────────────────────────────────────────────────
  • 10:15 – 10:30 AM (15 min) — Quick break / email triage
  • 11:00 AM – 12:45 PM (1h 45min) — ✅ Best block for deep work
  • 2:00 – 3:00 PM (1h) — Good for board deck review/follow-up

╚══════════════════════════════════════════════════════════════════╝
```

---

## Example Output: Post-Meeting Recap Email

```
Subject: Recap — Product Roadmap Review (April 8)

Hi all,

Thanks for a productive roadmap session this morning. Here's the
summary and action items.

SUMMARY
We aligned on Q2 priorities, agreed to reduce the target list from
14 to 10 initiatives, and decided to add 2 engineers to the auth
migration to get it back on track.

KEY DECISIONS
• Q2 roadmap reduced to 10 priority items (down from 14)
• Auth migration gets 2 additional engineers from Platform team
• API partner integration moved to Q3 pending their timeline
  confirmation

ACTION ITEMS
┌──────────────────────────────────┬────────────┬────────────┐
│ Task                             │ Owner      │ Due Date   │
├──────────────────────────────────┼────────────┼────────────┤
│ Publish final Q2 priority list   │ Mike T.    │ April 10   │
│ Confirm eng resource allocation  │ Lisa P.    │ April 11   │
│ Draft API partner communication  │ Sarah C.   │ April 12   │
│ Update board deck with Q2 plan   │ Mike T.    │ April 14   │
└──────────────────────────────────┴────────────┴────────────┘

NEXT STEPS
Roadmap review follow-up scheduled for April 22.
Mike will share the final priority list by Friday for async review.

Best,
[Executive Assistant Name]
```

---

## Implementation Options

| Approach | Complexity | Cost | Best For |
|----------|-----------|------|----------|
| **Zapier + Claude API** | Low | ~$50/mo | Solo EA, quick setup |
| **Make.com + Claude API + Notion** | Medium | ~$80/mo | Small teams, visual builder |
| **Python scripts + cron + Claude API** | Medium | ~$30/mo | Technical EAs, full control |
| **n8n (self-hosted) + Claude API** | High | ~$20/mo | Privacy-conscious orgs |

---

## Why This Should Be Implemented

**For the Executive Assistant:**
- Eliminates 2–3 hours of daily meeting prep research
- No more manually tracking action items across notebooks and emails
- Recap emails go from 20 minutes each to a 2-minute review
- Reduces "did I miss anything?" anxiety — the system catches everything

**For the Executive:**
- Shows up to every meeting fully briefed without effort
- Never drops a follow-up — action items are systematically tracked
- Gets a single daily document instead of scattered prep in 5 different tools
- Accountability improves because commitments are captured and visible

**ROI Calculation:**
- EA salary: ~$65K–$90K/year → ~$31–$43/hour
- 12 hours saved/week × $37/hour (midpoint) = **$23,000/year in recaptured productivity**
- Workflow cost: ~$50–$80/month = **$600–$960/year**
- **Net ROI: ~$22,000–$23,000/year per EA**

The recaptured time lets the EA focus on higher-value strategic support — vendor negotiations, event planning, team coordination — instead of copy-pasting between apps.

---

*Blueprint created: April 8, 2026 | Author: heymarii | Model: Claude Opus*
