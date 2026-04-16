---
name: google-calendar
description: >
  Read and create events in Google Calendar. Use when the user asks about their schedule,
  upcoming meetings, wants to add an event, or asks what they have planned. Never update
  or delete existing events — create only and read only.
---

# Google Calendar Skill

Interact with Google Calendar via the REST API using curl.

## Authentication

Get a fresh access token using the refresh token:
```bash
ACCESS_TOKEN=$(curl -s -X POST "https://oauth2.googleapis.com/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=$GOOGLE_CLIENT_ID" \
  -d "client_secret=$GOOGLE_CLIENT_SECRET" \
  -d "refresh_token=$GOOGLE_REFRESH_TOKEN" \
  -d "grant_type=refresh_token" | jq -r '.access_token')
```

Always fetch a fresh access token at the start of each Calendar operation. Never store or log the access token.

## List Calendars
```bash
curl -s "https://www.googleapis.com/calendar/v3/users/me/calendarList" \
  -H "Authorization: Bearer $ACCESS_TOKEN" | jq '.items[] | {id, summary, primary}'
```

## Get Upcoming Events
```bash
curl -s "https://www.googleapis.com/calendar/v3/calendars/primary/events" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -G \
  --data-urlencode "timeMin=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --data-urlencode "maxResults=20" \
  --data-urlencode "singleEvents=true" \
  --data-urlencode "orderBy=startTime" | jq '.items[] | {summary, start, end, location}'
```

## Create an Event

Always confirm with the user before creating. Show event details and ask for approval first.
```bash
curl -s -X POST "https://www.googleapis.com/calendar/v3/calendars/primary/events" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "summary": "Event Title",
    "description": "Optional description",
    "location": "Optional location",
    "start": {
      "dateTime": "2025-01-15T10:00:00+01:00",
      "timeZone": "Europe/Copenhagen"
    },
    "end": {
      "dateTime": "2025-01-15T11:00:00+01:00",
      "timeZone": "Europe/Copenhagen"
    }
  }' | jq '{id, summary, start, end, htmlLink}'
```

## Hard Restrictions

- Never call PUT, PATCH, or DELETE on existing events
- Never modify, reschedule, or cancel existing events
- Always confirm with user before creating an event
- Default timezone: Europe/Copenhagen
- Read and create only
