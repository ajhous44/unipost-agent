---
name: unipost
description: Schedule social media posts to LinkedIn profiles, X, TikTok, Facebook Pages (beta), and YouTube (beta) through UniPost's hosted MCP server. Every post an agent schedules waits for a human to approve it.
homepage: https://unipost.ai/agents
---

# UniPost

UniPost is an agent-first social media scheduler. You connect to its hosted MCP server, list the user's accounts, draft and schedule posts, and the user approves each post before it publishes. There is no CLI to install and no local daemon: everything is MCP over HTTPS.

- Server: `https://mcp.unipost.ai/mcp` (remote HTTP MCP, OAuth with PKCE, or an API key)
- Docs: https://unipost.ai/docs/mcp and https://unipost.ai/docs/agents/tools
- Machine-readable overview: https://unipost.ai/llms.txt

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

If the client cannot complete OAuth, a workspace owner or admin creates a key under https://unipost.ai/dashboard/api-keys and the client sends it as `Authorization: Bearer <key>`. Keep the key in the client's config or an environment variable; never paste it into a chat, a commit, or a post.

## Rules

1. **Nothing publishes without approval.** `schedule_post` always creates a group in `pending_approval` and returns an approval link. Show the user the full text, the destination accounts, the media, and the time, and call `approve_post` only after they say yes to that exact post. If you cannot see the user's answer, leave the post pending and give them the approval link.
2. **Use real account IDs.** Call `list_accounts` first and use the `id` values it returns. Skip accounts where `needs_reconnect` is true. Confirm the accounts belong to the workspace the user meant.
3. **Check X before posting there.** Call `get_x_allowance` first. A plain X post costs 1 credit and an X post containing a link costs 13. The free plan has 10 credits a month and paid plans 50 for each of the first 10 connected X accounts and 20 for each after; each X account can post 10 times an hour and 25 times a day. Do not add a link to an X post unless the user asked for one.
4. **Do not retry blindly.** `schedule_post` is not idempotent, so check `list_posts` before resending an uncertain request. Approval is not proof of publication: confirm with `list_posts` and read each destination's status and error.
5. **Media must be public HTTPS links.** Images: JPEG, PNG, GIF, or WebP up to 8 MB. Videos: MP4, MOV, or WebM up to 1 GB. Local files, authenticated links, and private network URLs do not work. Facebook Pages take text, images, or one video; LinkedIn profiles up to 9 images or one video; X up to 4 images or one video. Never mix a video with images in one post, and when posting to several accounts use media every selected platform accepts.
6. **Mind the budget.** The free plan includes 500 tool calls a month and paid plans 1,000 per account of capacity; requests are limited to 60 a minute per workspace. Reuse results instead of polling.
7. **Billing stays with the human.** There is no MCP billing tool and no payment endpoint. Buying accounts happens in the owner's signed-in browser; read https://unipost.ai/agent-checkout.md before helping with a purchase.

## Tools

### `list_accounts`

No arguments. Returns each publishable account's `id`, `platform`, `username`, `name`, `profile`, and `needs_reconnect`. Supported today: LinkedIn profiles, X, and TikTok. In beta: Facebook Pages (only Meta-approved users can connect until Meta's review passes) and YouTube (uploads private for now). Instagram is coming soon.

### `list_posts`

| Parameter | Type | Default | Values |
| --- | --- | --- | --- |
| `status` | string, optional | all | `pending_approval`, `scheduled`, `publishing`, `posted`, `failed` |
| `limit` | integer, optional | 20 | 1–50 |

Returns posts soonest first with post ID, group ID, content, status, scheduled and published times, destination, post URL, metrics when the platform reports them, an error message when one failed, and `approval_url` for pending posts. There is no pagination cursor.

### `schedule_post`

| Parameter | Type | Required | Rules |
| --- | --- | --- | --- |
| `account_ids` | array of UUID strings | yes | 1–10 distinct IDs from `list_accounts` |
| `content` | string | yes | 1–3,000 characters; a destination platform may allow less |
| `scheduled_for` | ISO 8601 date-time | yes | Include the timezone, e.g. `2026-10-06T09:00:00-05:00`; now or later, within one year |
| `media_urls` | array of URL strings | no | Up to 10 public HTTPS image or video links |

Creates one group across all destinations in `pending_approval` and returns an approval link. Example arguments:

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

Approves the whole group the post belongs to. It publishes at its scheduled time, or right away if that time has passed. Works only for posts created by this connection's user; posts created by another person's agent, and any group that includes TikTok, are approved through the approval link in UniPost.

### `cancel_post`

| Parameter | Type | Required |
| --- | --- | --- |
| `post_id` | UUID string | yes |

Deletes the unpublished posts in the group (`pending_approval`, `scheduled`, or `failed`). Posts that are `publishing` or `posted` cannot be canceled.

### `get_analytics`

| Parameter | Type | Default | Range |
| --- | --- | --- | --- |
| `days` | integer, optional | 30 | 1–365 |

Views, likes, comments, and shares totaled by platform, plus the top posts by views. LinkedIn profiles and X do not report post metrics to UniPost yet, so a missing number is not zero engagement.

### `get_x_allowance`

No arguments. Remaining monthly X credits for the owner and each X account's remaining hourly and daily posting allowance.

## Workflow

1. `list_accounts`. Tell the user which accounts you found and which need reconnecting.
2. Draft the post. Show the text, destinations, media links, and time in the user's timezone.
3. `schedule_post`. Return the approval link.
4. When the user says yes to that post, `approve_post`. Otherwise `cancel_post` or leave it pending.
5. `list_posts` with `status: "posted"` to confirm, and `get_analytics` later for results.

## Example prompts

- "List the social accounts in this workspace."
- "Prepare a post for tomorrow at 9 a.m. in my timezone. Show me the full text and destination accounts before you publish it."
- "Schedule this video to the Facebook Page and my LinkedIn profile on Tuesday at 9 a.m. Eastern, then send me the approval link."
- "What's left of my X allowance this month?"
- "Which posts did best in the last 30 days?"

## Pricing

The first two connected accounts are free. After that it's $3, $1, and $0.50 per account a month as the user adds more. MCP access, approvals, analytics, and unlimited teammates come with every account, including the free one. Details: https://unipost.ai/#pricing.
