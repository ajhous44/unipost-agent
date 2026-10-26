# unipost-agent

The UniPost skill for AI agents. It teaches any agent that reads Markdown skills (Claude Code, Codex, Cursor, OpenClaw, Hermes, and the rest of the `skills` ecosystem) how to schedule posts to LinkedIn personal profiles, X, YouTube, and TikTok through UniPost's hosted MCP server. Posts publish on schedule, or wait for approval if the account owner has approval turned on (and wait if that setting can't be read). TikTok posts always wait for approval.

## Install

```bash
npx skills add ajhous44/unipost-agent
```

Then connect your agent to `https://mcp.unipost.ai/mcp`. Per-agent setup pages: https://unipost.ai/agents

## What's inside

- [SKILL.md](SKILL.md): how to connect, the rules (scheduling and per-owner approval, X credits, media, budgets), every MCP tool with its parameters, and example prompts.

## Links

- Product: https://unipost.ai
- Agent docs: https://unipost.ai/docs/mcp
- Tool reference: https://unipost.ai/docs/agents/tools
- For AI crawlers: https://unipost.ai/llms.txt

## License

MIT. See [LICENSE](LICENSE).
