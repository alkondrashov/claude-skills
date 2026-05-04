---
name: meta-pages
description: Read Facebook Page inbox conversations and messages via the Pages Messaging API. Use for qualitative analysis of customer conversations, drop-off reasons, and message content.
---

# Meta Pages Messaging

Read Facebook Page inbox conversations and messages via the Pages Messaging API.

This is separate from `meta-ads-insights` (Marketing API). Use this skill when you need actual conversation content, not aggregate ad metrics.

## Credentials

Stored as env vars (already set in `~/.claude/settings.json`):

- `META_PAGE_ACCESS_TOKEN` — User Access Token with `pages_messaging` + `pages_read_engagement` scopes

## Known Pages

| Page | ID |
|---|---|
| WPG | `100804966047864` |

## API Base URL

```
https://graph.facebook.com/v21.0/
```

## Getting a Page Access Token

`META_PAGE_ACCESS_TOKEN` is a **User** Access Token. The conversations API requires a **Page** Access Token. Exchange it at the start of every session:

```bash
curl -s "https://graph.facebook.com/v21.0/me/accounts?access_token=$META_PAGE_ACCESS_TOKEN"
```

This returns a list of pages with their `access_token` values. Use the token for the relevant page (e.g. WPG `100804966047864`).

To check which permissions the current token has:

```bash
curl "https://graph.facebook.com/v21.0/me/permissions?access_token=$META_PAGE_ACCESS_TOKEN"
```

Required permissions: `pages_messaging`, `pages_read_engagement`.

If `pages_messaging` is missing, regenerate the token via [developers.facebook.com/tools/explorer](https://developers.facebook.com/tools/explorer/) with the correct scopes, then re-exchange via `/me/accounts`.

## Common Operations

### List conversations

```bash
PAGE_TOKEN="<token from /me/accounts>"
PAGE_ID="100804966047864"

curl "https://graph.facebook.com/v21.0/$PAGE_ID/conversations?fields=id,snippet,updated_time,message_count,participants&limit=100&access_token=$PAGE_TOKEN"
```

Filter by month in Python:

```python
april = [c for c in data["data"] if c["updated_time"].startswith("2026-04")]
```

### Read messages in a conversation

```bash
curl "https://graph.facebook.com/v21.0/{conversation_id}/messages?fields=message,from,created_time&limit=50&access_token=$PAGE_TOKEN"
```

Messages are returned newest-first — reverse the list to read chronologically:

```python
messages = list(reversed(data["data"]))
for m in messages:
    print(m["from"]["name"], m.get("message", "[media]"))
```

### Fetch full thread for a batch of conversations

```bash
for ID in t_xxx t_yyy t_zzz; do
  echo "===== $ID ====="
  curl -s "https://graph.facebook.com/v21.0/$ID/messages?fields=message,from,created_time&limit=50&access_token=$PAGE_TOKEN" | python3 -c "
import json,sys
d=json.load(sys.stdin)
for m in reversed(d.get('data',[])):
    print(f'  [{m[\"created_time\"][5:16]}] {m[\"from\"][\"name\"]}: {m.get(\"message\",\"[media]\")[:120]}')
"
done
```

## Conversation Fields Reference

| Field | Meaning |
|---|---|
| `id` | Conversation ID (prefix `t_`) |
| `snippet` | Last message preview |
| `updated_time` | Time of last message |
| `message_count` | Total messages in thread |
| `participants` | Array of participant objects (`name`, `id`) |

## Notes

- `[media]` in output means the message was an image, video, or sticker (no `message` field)
- The page itself appears as a participant with `id` = the page ID
- Conversations are paginated — use the `paging.next` cursor for >100 results
