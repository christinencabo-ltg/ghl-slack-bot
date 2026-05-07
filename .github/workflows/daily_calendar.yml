import os
import requests
from datetime import datetime, timezone, timedelta
from zoneinfo import ZoneInfo

GHL_API_KEY = os.environ["GHL_API_KEY"]
GHL_LOCATION_ID = os.environ["GHL_LOCATION_ID"]
SLACK_WEBHOOK_URL = os.environ["SLACK_WEBHOOK_URL"]

DISCOVERY_CALL_CALENDAR_ID = "ZOKEQ34FlaJWERsNXFqR"
FINAL_INTERVIEW_CALENDAR_ID = "RoqvlMuoLKH2RBn4WtyZ"

ET = ZoneInfo("America/New_York")

HEADERS = {
    "Authorization": f"Bearer {GHL_API_KEY}",
    "Version": "2021-04-15",
    "Content-Type": "application/json",
}

contact_cache = {}


def get_week_range():
    """Returns timestamps for Monday 12:00am to Friday 11:59pm of the current week in ET."""
    now = datetime.now(ET)
    # Find Monday of current week
    monday = now - timedelta(days=now.weekday())
    monday = monday.replace(hour=0, minute=0, second=0, microsecond=0)
    # Friday end of day
    friday = monday + timedelta(days=4)
    friday = friday.replace(hour=23, minute=59, second=59, microsecond=0)
    return int(monday.timestamp() * 1000), int(friday.timestamp() * 1000)


def fetch_events(calendar_id, start_time, end_time):
    url = "https://services.leadconnectorhq.com/calendars/events"
    params = {
        "locationId": GHL_LOCATION_ID,
        "calendarId": calendar_id,
        "startTime": start_time,
        "endTime": end_time,
    }
    resp = requests.get(url, headers=HEADERS, params=params)
    if resp.status_code == 200:
        return resp.json().get("events", [])
    return []


def get_contact_name(contact_id):
    if not contact_id:
        return "Unknown"
    if contact_id in contact_cache:
        return contact_cache[contact_id]
    url = f"https://services.leadconnectorhq.com/contacts/{contact_id}"
    resp = requests.get(url, headers=HEADERS)
    if resp.status_code == 200:
        data = resp.json().get("contact", {})
        name = data.get("name") or f"{data.get('firstName', '')} {data.get('lastName', '')}".strip()
        contact_cache[contact_id] = name or "Unknown"
    else:
        contact_cache[contact_id] = "Unknown"
    return contact_cache[contact_id]


def to_et(dt_str):
    dt = datetime.fromisoformat(dt_str)
    return dt.astimezone(ET)


def format_schedule(start_str, end_str):
    """Format as: Fri, May 1, 2026, 12:00 pm - 12:45 pm (EDT)"""
    start_dt = to_et(start_str)
    end_dt = to_et(end_str)
    tz_abbr = start_dt.strftime("%Z")
    date_part = start_dt.strftime("%a, %b %-d, %Y")
    start_time = start_dt.strftime("%-I:%M %p").replace("AM", "am").replace("PM", "pm")
    end_time = end_dt.strftime("%-I:%M %p").replace("AM", "am").replace("PM", "pm")
    return f"{date_part}, {start_time} - {end_time} ({tz_abbr})"


def format_datetime(dt_str):
    """Format as: Mon, Apr 06 2026, 12:00 pm (EDT)"""
    dt = to_et(dt_str)
    tz_abbr = dt.strftime("%Z")
    return dt.strftime("%a, %b %-d %Y, %-I:%M %p").replace("AM", "am").replace("PM", "pm") + f" ({tz_abbr})"


def is_real_appointment(event):
    source = event.get("createdBy", {}).get("source", "")
    title = event.get("title", "").lower()
    if source == "google_calendar":
        return "interview" in title or "discovery call" in title
    return True


def is_generic_va_entry(title):
    """Returns True if title is generic 'VA <> Client Interview' (no specific VA name)."""
    if "<>" in title:
        va_part = title.split("<>")[0].strip().lower()
        return va_part == "va"
    return False


def build_slack_message():
    start_ms, end_ms = get_week_range()
    now = datetime.now(timezone.utc).astimezone(ET)
    today = now.strftime("%A, %B %-d %Y")

    # Week label: Mon May 5 - Fri May 9, 2026
    monday = now - timedelta(days=now.weekday())
    friday = monday + timedelta(days=4)
    week_label = f"{monday.strftime('%a %b %-d')} - {friday.strftime('%a %b %-d, %Y')}"

    # Monday = start of week, Friday = end of week
    if now.weekday() == 0:
        period_label = "Week Ahead"
    elif now.weekday() == 4:
        period_label = "End of Week Summary"
    else:
        period_label = "This Week"

    lines = [
        "*GHL Calendar Update*",
        f"{today}  |  {period_label}: {week_label}",
        "----------------------------------------",
    ]

    # --- Discovery Calls ---
    dc_events = fetch_events(DISCOVERY_CALL_CALENDAR_ID, start_ms, end_ms)
    dc_events = [e for e in dc_events if is_real_appointment(e)]
    dc_events.sort(key=lambda e: e.get("startTime", ""))

    lines.append(f"\nLTG: Discovery Calls — {len(dc_events)} booked this week")
    if dc_events:
        for event in dc_events:
            client_name = get_contact_name(event.get("contactId", ""))
            date_time = format_datetime(event.get("startTime", ""))
            lines.append(f"  • Client: {client_name} on {date_time}")
    else:
        lines.append("  No discovery calls found")

    lines.append("")

    # --- Final Interviews ---
    fi_events = fetch_events(FINAL_INTERVIEW_CALENDAR_ID, start_ms, end_ms)
    fi_events = [e for e in fi_events if is_real_appointment(e)]

    # Separate clients (generic) and VAs (named)
    clients = [e for e in fi_events if is_generic_va_entry(e.get("title", ""))]
    vas = [e for e in fi_events if not is_generic_va_entry(e.get("title", ""))]

    total = len(fi_events)
    lines.append(f"Final Interviews: VA <> Client — {total} scheduled this week")

    if fi_events:
        # Group by date
        from collections import defaultdict
        by_date = defaultdict(lambda: {"clients": [], "vas": []})

        for e in clients:
            date_key = to_et(e["startTime"]).strftime("%Y-%m-%d")
            by_date[date_key]["clients"].append(e)
        for e in vas:
            date_key = to_et(e["startTime"]).strftime("%Y-%m-%d")
            by_date[date_key]["vas"].append(e)

        sorted_dates = sorted(by_date.keys())
        for i, date_key in enumerate(sorted_dates):
            group = by_date[date_key]
            # Sort within each group by start time
            group["clients"].sort(key=lambda e: e.get("startTime", ""))
            group["vas"].sort(key=lambda e: e.get("startTime", ""))

            # Add divider between date groups (not before the first one)
            if i > 0:
                lines.append("\n----------------------------------------")

            # Clients first
            for event in group["clients"]:
                contact_name = get_contact_name(event.get("contactId", ""))
                schedule = format_schedule(event.get("startTime", ""), event.get("endTime", ""))
                lines.append(f"\n*Client Name:* {contact_name}")
                lines.append(f"*Schedule:* {schedule}")

            # Then VAs
            for event in group["vas"]:
                contact_name = get_contact_name(event.get("contactId", ""))
                schedule = format_schedule(event.get("startTime", ""), event.get("endTime", ""))
                lines.append(f"\n*VA Name:* {contact_name}")
                lines.append(f"*Schedule:* {schedule}")
    else:
        lines.append("  No final interviews found")

    return "\n".join(lines)


def post_to_slack(message):
    payload = {"text": message}
    resp = requests.post(SLACK_WEBHOOK_URL, json=payload)
    resp.raise_for_status()
    print("Posted to Slack successfully.")


if __name__ == "__main__":
    message = build_slack_message()
    post_to_slack(message)
