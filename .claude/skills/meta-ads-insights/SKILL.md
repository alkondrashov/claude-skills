---
name: meta-ads-insights
description: Query Meta Ads campaigns, messaging stats, and lead counts via the Facebook Marketing API
---

# Meta Ads Insights

Query Meta Ads campaigns, messaging stats, and lead counts via the Facebook Marketing API.

## Credentials

Stored as env vars (already set in `~/.claude/settings.json`):

- `META_APP_ID`
- `META_APP_SECRET`
- `META_ACCESS_TOKEN`
- `META_AD_ACCOUNT_ID`

## API Base URL

```
https://graph.facebook.com/v21.0/
```

## Common Operations

### Get an access token

Ad account data requires a **user access token** with `ads_read` permission. App access tokens (client_credentials) will not work for campaign or insights queries.

Generate a user access token via the [Meta Token Tools](https://developers.facebook.com/tools/explorer/) with `ads_read` scope, then update `META_ACCESS_TOKEN` in `~/.claude/settings.json`.

To check if the current token is valid:

```bash
curl "https://graph.facebook.com/v21.0/me?access_token=$META_ACCESS_TOKEN"
```

### List all campaigns

```bash
curl "https://graph.facebook.com/v21.0/$META_AD_ACCOUNT_ID/campaigns?fields=id,name,status,objective,created_time&access_token=$META_ACCESS_TOKEN"
```

### Get insights for a campaign

```bash
curl "https://graph.facebook.com/v21.0/{campaign_id}/insights?fields=actions&date_preset=maximum&access_token=$META_ACCESS_TOKEN"
```

### Get campaign-level insights for a date range (all campaigns)

```bash
curl "https://graph.facebook.com/v21.0/$META_AD_ACCOUNT_ID/insights?fields=campaign_name,actions,spend,impressions,reach&time_range=%7B%22since%22%3A%222026-04-01%22%2C%22until%22%3A%222026-04-30%22%7D&level=campaign&access_token=$META_ACCESS_TOKEN"
```

Use `level=campaign` to break results down per campaign. Replace `time_range` dates as needed (URL-encoded JSON).

### Key messaging action types

| Action type | Meaning |
|---|---|
| `onsite_conversion.total_messaging_connection` | Total messaging connections |
| `onsite_conversion.messaging_conversation_started_7d` | Conversations started (7-day) |
| `onsite_conversion.messaging_conversation_replied_7d` | Conversations replied (7-day) |
| `onsite_conversion.messaging_first_reply` | First replies from business |
| `onsite_conversion.messaging_user_depth_2_message_send` | Reached 2nd exchange |
| `onsite_conversion.messaging_user_depth_3_message_send` | Reached 3rd exchange |
| `onsite_conversion.messaging_user_depth_5_message_send` | Reached 5th exchange |
| `onsite_conversion.lead` | Leads generated |

## Python Usage (facebook-business SDK)

```python
import os
from facebook_business.api import FacebookAdsApi
from facebook_business.adobjects.adaccount import AdAccount
from facebook_business.adobjects.campaign import Campaign

app_id     = os.environ["META_APP_ID"]
app_secret = os.environ["META_APP_SECRET"]
access_token = os.environ["META_ACCESS_TOKEN"]
ad_account_id = os.environ["META_AD_ACCOUNT_ID"]

FacebookAdsApi.init(app_id, app_secret, access_token)

account = AdAccount(ad_account_id)

# List campaigns
campaigns = account.get_campaigns(fields=["id", "name", "status", "objective", "created_time"])
for c in campaigns:
    print(c["id"], c["name"], c["status"])

# Get insights for a specific campaign
campaign = Campaign(campaign_id)
insights = campaign.get_insights(
    fields=["actions"],
    params={"date_preset": "maximum"},
)

# Parse messaging actions
for insight in insights:
    for action in insight.get("actions", []):
        if action["action_type"].startswith("onsite_conversion"):
            print(action["action_type"], action["value"])
```

Install the SDK:

```bash
pip install facebook-business
```
