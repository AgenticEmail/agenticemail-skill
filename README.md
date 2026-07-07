# AgenticEmail skill

An [agent skill](https://www.skills.sh/) that teaches an AI agent to give itself
a real email inbox with [AgenticEmail](https://agenticemail.dev): create
inboxes at runtime, send and receive email, reply, and react to inbound mail,
over a REST API, TypeScript/Python SDKs, a CLI, or a hosted MCP server, with
optional end-to-end encryption.

## Install

```bash
npx skills add AgenticEmail/agenticemail-skill
```

## What the agent can do

- Create an addressable inbox on demand.
- Send and reply to email.
- Receive inbound mail as parsed JSON via webhook or WebSocket.
- Wait for a reply (verification and OTP loops).
- Turn on opt-in, zero-knowledge, agent-to-agent end-to-end encryption.

## When it triggers

When an agent needs an email address, not just the ability to send: signing up
for a service, receiving a verification or OTP code, getting notified, running a
support or outreach workflow, or holding an email conversation with a person or
another agent.

## Contents

- `SKILL.md` - the skill the agent loads.
- `references/reference.md` - REST, SDK, CLI, and MCP details.

## Links

- Product: https://agenticemail.dev
- Docs: https://agenticemail.dev/docs
- Hosted MCP server: `https://api.agenticemail.dev/mcp`
- Get an API key: https://app.agenticemail.dev

## License

Apache-2.0
