---
name: unipost
description: Schedule social media posts to LinkedIn personal profiles, X, YouTube, and TikTok through UniPost's hosted MCP server. Posts publish on schedule, or wait for approval if the account owner has approval turned on. TikTok posts always wait for approval.
homepage: https://unipost.ai/agents
---

# UniPost

UniPost is an agent-first social media scheduler. You connect to its hosted MCP server, list the user's accounts, and schedule posts. Posts publish at their scheduled time, or wait for approval if the account owner has approval turned on. There is no CLI to install and no local daemon. Everything runs as MCP over HTTPS.

- Server: `https://mcp.unipost.ai/mcp` (remote HTTP MCP, OAuth with PKCE, or an API key)
- Docs: https://unipost.ai/docs/mcp and https://unipost.ai/docs/agents/tools
- Machine-readable overview: https://unipost.ai/llms.txt

This skill covers four destinations: LinkedIn personal profiles, X, YouTube video uploads, and TikTok videos. If `list_accounts` returns another platform, tell the user this skill doesn't cover it and don't schedule to it unless they ask.

## Connect

Add the server URL as a remote HTTP MCP server in your client, then sign in when the browser opens. The user picks the workspace you may use and selects **Allow**. Only workspace owners and admins can authorize an agent.

| Client | How |
| --- | --- |
| Claude Code | `claude mcp add --transport http unipost https://mcp.unipost.ai/mcp`, then `/mcp` → UniPost → sign in |
| Codex | Add `[mcp_servers.unipost]` with `url = "https://mcp.unipost.ai/mcp"` to `~/.codex/config.toml`, then `codex mcp login unipost` |
| Cursor, Gemini CLI, other HTTP MCP clients | Add the URL as an HTTP MCP server and complete the browser sign-in |
| Claude web, desktop, mobile | https://claude.ai/directory/unipost → Connect |
| ChatGPT | Custom MCP app with OAuth sign-in (developer mode under Settings → Apps → Advanced settings) |
| Grok, Grok Bot | Custom connector (Grok) or custom MCP server under Plugins (Grok Bot) |

Step-by-step pages: https://unipost.ai/agents/claude-code, /agents/codex, /agents/cursor, /agents/chatgpt, /agents/claude, /agents/grok-bot, /agents/gemini-cli.

If the client can't complete OAuth, a workspace owner or admin creates a key at https://unipost.ai/dashboard/api-keys and the client sends it as `Authorization: Bearer <key>`. Keep the key in the client's config or an environment variable. Never paste it into a chat, a commit, or a post.

## Rules

1. **Read the status `schedule_post` returns.** Approval is a per-owner setting on the workspace owner's profile. Accounts that existed before this setting shipped have it on. New accounts have it off. TikTok posts always wait for approval, whatever the setting.
   - `scheduled`: the post publishes at `scheduled_for`. Confirm the time to the user.
   - `pending_approval`: the post waits. The result includes `approval_url`. Give the user that link, or call `approve_post` after they say yes to that exact post.
   - Don't guess which one you'll get. If UniPost can't read the owner's setting, the post waits for approval.
2. **Show the post before you schedule it.** A `scheduled` post publishes without another check, so show the user the full text, destinations, media, and time first, and schedule only after they agree. Never approve a post the user hasn't seen and agreed to. If you can't see their answer, leave it pending and give them the approval link.
3. **Use real account IDs.** Call `list_accounts` first and use the `id` values it returns. Skip accounts where `needs_reconnect` is true. Confirm the accounts belong to the workspace the user meant.
4. **Check X credits before posting there.** Call `get_x_allowance` first. A plain X post costs 1 credit. A post with a link costs 13, and a bare domain such as `example.com` counts as a link. The free plan has 10 credits a month, so on the free plan a link post won't go to X. When that happens `schedule_post` still saves the post and returns a `warning`; tell the user and offer to remove the link. Paid plans get 50 credits a month for each of the first 10 connected X accounts and 20 for each after. Each X account can post at most 10 times an hour and 25 times a day. Posts over these limits fail when they come due. Don't add a link to an X post unless the user asked for one.
5. **One post per account, no threads.** `schedule_post` sends the same text and media to every account in `account_ids`. It can't create X threads or replies.
6. **Don't retry blindly.** `schedule_post` is not idempotent, so check `list_posts` before resending an uncertain request. A `scheduled` or approved status is not proof of publication. Confirm with `list_posts` and read each destination's status and error.
7. **Media must be public HTTPS links.** Images: JPEG, PNG, GIF, or WebP up to 8 MB. Videos: MP4, MOV, or WebM up to 1 GB. The host must send a Content-Length header. Local files, authenticated links, and private network URLs don't work. Per destination:
   - LinkedIn profiles: text, up to 9 images, or one video.
   - X: text, up to 4 images, or one video.
   - YouTube: exactly one video, no images. The first line of `content` becomes the title (cut at 100 characters).
   - TikTok: exactly one video, no images. The user picks privacy at the approval link, and TikTok may keep the post private until UniPost's app review is done.
   Never mix a video with images. When posting to several accounts, use media every selected platform accepts. If the media doesn't fit, `schedule_post` returns an error saying why and saves nothing.
8. **Mind the budget.** The free plan includes 500 tool calls a month and paid plans 1,000 per account of capacity. Requests are limited to 60 a minute per workspace. Reuse results instead of polling.
9. **Billing stays with the human.** There is no MCP billing tool and no payment endpoint. Buying accounts happens in the owner's signed-in browser. Read https://unipost.ai/agent-checkout.md before helping with a purchase.

## Tools

### `list_accounts`

No arguments. Returns each publishable account's `id`, `platform`, `username`, `name`, `profile`, and `needs_reconnect`.

### `list_posts`

| Parameter | Type | Default | Values |
| --- | --- | --- | --- |
| `status` | string, optional | all | `pending_approval`, `scheduled`, `publishing`, `posted`, `sent_to_inbox`, `failed` |
| `limit` | integer, optional | 20 | 1 to 50 |

Returns posts ordered by scheduled time, earliest first, with post ID, group ID, content, status, scheduled and published times, destination, post URL, metrics when the platform reports them, an error message when one failed, and `approval_url` for pending posts. `sent_to_inbox` means a TikTok video went to the creator's TikTok inbox as a draft. There is no pagination cursor, so filter by `status` to find recent posts.

### `schedule_post`

| Parameter | Type | Required | Rules |
| --- | --- | --- | --- |
| `account_ids` | array of UUID strings | yes | 1 to 10 distinct IDs from `list_accounts` |
| `content` | string | yes | 1 to 3,000 characters; a destination platform may allow less |
| `scheduled_for` | ISO 8601 date-time | yes | Include the timezone, e.g. `2026-10-06T09:00:00-05:00`; now or later, within one year |
| `media_urls` | array of URL strings | no | Up to 10 public HTTPS image or video links |
| `tiktok_inbox_draft` | boolean | no | TikTok only. Sends the video to the creator's TikTok inbox as a draft instead of publishing it, so they can finish it (sound, caption, privacy) in the TikTok app. Needs exactly one video. If the TikTok connection lacks upload permission, the error tells the user to reconnect TikTok. |

TikTok (Business) accounts take extra parameters (`music_sound_id` is required for them). Read the tool's schema before scheduling to one, and schedule TikTok and TikTok (Business) accounts in separate posts.

Creates one group across all destinations and returns `post_group_id`, `status`, and the created posts. The status is `scheduled` when the owner has approval off and no TikTok account is included. It is `pending_approval`, with `approval_url`, when the owner has approval on or the group includes TikTok. A free-plan link post to X also returns a `warning`. Example arguments:

```json
{
  "account_ids": ["11111111-1111-4111-8111-111111111111"],
  "content": "Our latest update is ready. Here's what we built this week.",
  "scheduled_for": "2026-10-06T09:00:00-05:00"
}
```

### `approve_post`

| Parameter | Type | Required |
| --- | --- | --- |
| `post_id` | UUID string | yes |

Approves a `pending_approval` post and the rest of its group. It publishes at its scheduled time, or right away if that time has passed. Works only for posts created by this connection's user. Posts created by another person's agent, and any group that includes TikTok, are approved through the approval link, where TikTok's settings are chosen.

### `cancel_post`

| Parameter | Type | Required |
| --- | --- | --- |
| `post_id` | UUID string | yes |

Deletes the unpublished posts in the group (`pending_approval`, `scheduled`, or `failed`). Posts that are `publishing`, `posted`, or `sent_to_inbox` can't be canceled. Delete those on the platform or in the TikTok app.

### `get_analytics`

| Parameter | Type | Default | Range |
| --- | --- | --- | --- |
| `days` | integer, optional | 30 | 1 to 365 |

Views, likes, comments, and shares for published posts, totaled by platform, plus the top posts by views. Numbers refresh every few hours for a month after posting. LinkedIn profiles and X don't report post metrics to UniPost yet, so a missing number is not zero engagement.

### `get_x_allowance`

No arguments. Returns X credits used, the monthly quota, credits remaining and when they reset, the cost of a plain post and a link post, and each X account's posts and remaining allowance for the last hour and last 24 hours.

## Workflow

1. `list_accounts`. Tell the user which accounts you found and which need reconnecting.
2. For X, call `get_x_allowance`.
3. Draft the post. Show the text, destinations, media links, and time in the user's timezone, and wait for a yes.
4. `schedule_post`. If the result is `scheduled`, confirm the time. If it is `pending_approval`, give the user the approval link. Pass on any `warning`.
5. When a post is pending and the user says yes to that post, `approve_post` (not for TikTok). Otherwise `cancel_post` or leave it pending.
6. After the scheduled time, `list_posts` with `status: "posted"` to confirm, and `get_analytics` later for results.

## Example prompts

- "List the social accounts in this workspace."
- "Draft a post for tomorrow at 9 a.m. in my timezone. Show me the full text and destination accounts before you schedule it."
- "Schedule this video to my YouTube channel and TikTok on Tuesday at 9 a.m. Eastern."
- "What's left of my X allowance this month?"
- "Which posts did best in the last 30 days?"

## Pricing

The first two connected accounts are free. After that it's $3, $1, and $0.50 per account a month as the user adds more. Details: https://unipost.ai/#pricing.
