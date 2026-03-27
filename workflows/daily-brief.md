# Daily Brief Workflow

**Purpose:** Generate a personalized morning briefing combining news, calendar, and tasks  
**Trigger:** Daily at 7am (via cron or scheduled task)  
**Output:** Markdown summary delivered to email or Notion

---

## Workflow Steps

```
Step 1: Gather Inputs
  ├── Fetch top 5 AI/tech news headlines (web search)
  ├── Read today's calendar events (calendar API)
  └── Read open tasks (task manager API)

Step 2: Prioritize
  └── Claude ranks tasks by urgency + importance
      Context: today's meetings inform priority

Step 3: Generate Brief
  └── Claude writes the briefing using this template:
      - 🌅 Good morning, {name}!
      - 📰 Top 3 things happening in tech today
      - 📅 Your day at a glance (meetings)
      - ✅ Top 3 tasks to focus on
      - 💡 One interesting idea or prompt for the day

Step 4: Deliver
  └── Send to email / post to Notion / push notification
```

## Prompt for Step 3

```
You are a personal assistant writing a morning briefing. Be concise, warm, and practical.

<news>
{NEWS_ITEMS}
</news>

<calendar>
{CALENDAR_EVENTS}
</calendar>

<tasks>
{OPEN_TASKS}
</tasks>

Write a morning brief following this format:
- Greet me warmly with today's date
- Summarize the 2-3 most interesting/relevant news items in 1 sentence each
- List my meetings for today with times
- Highlight my top 3 priority tasks based on deadlines and meeting context
- End with one thought-provoking question or idea to consider today

Keep the whole thing under 300 words. Use emoji sparingly but effectively.
```

## Implementation Notes

- Works great as a GitHub Actions workflow on a cron schedule
- Can use Claude API directly for the generation step
- Calendar integration: Google Calendar API or iCal parsing
- Task integration: Todoist, Linear, or Notion API
