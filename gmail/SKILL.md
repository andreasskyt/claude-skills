---
name: gmail
description: >
  Read emails from Gmail. Use when the user asks about their emails, wants to find a specific
  email, check unread messages, or search their inbox. Read only — no sending, no drafts,
  no modifications.
---

# Gmail Skill

Read Gmail via the REST API using curl. Read only — no write operations of any kind.

## Authentication

Get a fresh access token:
```bash
ACCESS_TOKEN=$(curl -s -X POST "https://oauth2.googleapis.com/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=$GOOGLE_CLIENT_ID" \
  -d "client_secret=$GOOGLE_CLIENT_SECRET" \
  -d "refresh_token=$GOOGLE_REFRESH_TOKEN" \
  -d "grant_type=refresh_token" | jq -r '.access_token')
```

Always fetch a fresh access token at the start of each Gmail operation. Never store or log it.

## List Unread Emails
```bash
curl -s "https://gmail.googleapis.com/gmail/v1/users/me/messages" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -G \
  --data-urlencode "q=is:unread" \
  --data-urlencode "maxResults=10" | jq '.messages[] | .id'
```

## Read an Email
```bash
curl -s "https://gmail.googleapis.com/gmail/v1/users/me/messages/{message_id}" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -G \
  --data-urlencode "format=full" | jq '{
    subject: (.payload.headers[] | select(.name=="Subject") | .value),
    from: (.payload.headers[] | select(.name=="From") | .value),
    date: (.payload.headers[] | select(.name=="Date") | .value)
  }'
```

## Search Emails
```bash
curl -s "https://gmail.googleapis.com/gmail/v1/users/me/messages" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -G \
  --data-urlencode "q=from:someone@example.com subject:invoice" \
  --data-urlencode "maxResults=10" | jq
```

Gmail search operators: from:, to:, subject:, is:unread, is:starred, after:2025/01/01, has:attachment

## Hard Restrictions

- Never call POST, PUT, PATCH, or DELETE on any Gmail endpoint
- Never send, draft, modify, delete, or label emails
- Read only — GET requests only
