---
name: agenticemail
description: Give an AI agent its own real email inbox with AgenticEmail. Create addressable inboxes at runtime, send and receive email, reply, and react to inbound mail, over a REST API, TypeScript/Python SDKs, a CLI, or a hosted MCP server, with optional end-to-end encryption. Use whenever an agent needs an email address to sign up for services, receive verification or OTP codes, get notifications, run support or outreach, or hold email conversations with people or other agents.
license: Apache-2.0
---

# AgenticEmail: email for AI agents

AgenticEmail gives an agent a real, addressable email inbox it owns. Unlike an
outbound-only sending API, every inbox both sends and receives: inbound mail
arrives as parsed JSON through a webhook or a WebSocket stream. Reach for this
whenever an agent needs to be reachable at an email address, not just able to
send.

Common triggers: the agent must sign up for a third-party service, receive a
verification or OTP code, get notified by email, run a support or outreach
workflow, or exchange email with a human or another agent.

## Setup

1. Get an API key (it starts with `am_`) at https://app.agenticemail.dev. The
   free tier includes real inboxes, sending, receiving, and webhooks.
2. Export it: `export AGENTICEMAIL_API_KEY=am_...`
3. Pick an integration path below. All of them call `https://api.agenticemail.dev`.

## Pick a path

- **Hosted MCP server (recommended for MCP-capable agents like Claude, Cursor,
  and the Vercel AI SDK).** Connect `https://api.agenticemail.dev/mcp` with the
  API key as a Bearer token. The agent then gets these tools directly:
  `create_inbox`, `list_inboxes`, `list_threads`, `get_thread`, `list_messages`,
  `get_message`, `send_message`, `reply_to_message`.
- **CLI** (`npm i -g agenticemail-cli`), for shell-driven agents.
- **SDK** (`npm i agenticemail` for TypeScript, or `pip install 'agenticemail[e2e]'`
  for Python), for code.

## Core flow: create, send, receive, reply

CLI:

```bash
# 1. give the agent an inbox
INBOX=$(agenticemail inboxes create | jq -r .id)

# 2. send
agenticemail messages send "$INBOX" --to user@example.com \
  --subject "Hello" --text "Sent by an agent."

# 3. read the inbox (parsed JSON)
agenticemail messages list "$INBOX" --limit 5

# 4. block until a reply arrives (great for OTP / verification loops)
agenticemail wait-for-message "$INBOX" --timeout 300
```

TypeScript SDK:

```ts
import { AgenticEmail } from "agenticemail";
const client = new AgenticEmail({ apiKey: process.env.AGENTICEMAIL_API_KEY! });

const inbox = await client.inboxes.create({ username: "assistant" });
await client.messages.send(inbox.id, {
  to: ["user@example.com"],
  subject: "Hello",
  text: "Sent by an agent.",
});
const { data } = await client.messages.list(inbox.id, { limit: 5 });
```

## Receiving inbound mail

Register a webhook so new mail reaches the agent the moment it arrives:

```bash
agenticemail webhooks create --url https://your-app/hook \
  --event-types message.received
```

Each event carries the parsed sender, subject, text, HTML, and attachments. A
WebSocket stream is available for the same events if the agent is already
running.

## End-to-end encryption (optional)

For agent-to-agent mail, AgenticEmail offers opt-in end-to-end encryption: the
private key is generated and kept on the client (SDK or CLI), never on the
server, so the platform cannot read the content. Enable it in the CLI with
`agenticemail keys generate <inbox>` then `agenticemail keys publish <inbox>`,
after which `messages send` and `messages get` encrypt and decrypt
automatically. In the SDK, pass an `e2e` identity when constructing the client.

Be honest about the boundary: encryption is opt-in and agent-to-agent only. It
works between two AgenticEmail inboxes that have each published a key; sends to
an external address like Gmail fall back to plaintext, and plaintext is the
default.

## Full reference

REST endpoints, SDK method signatures, the MCP config block, and the encryption
details are in `references/reference.md`. The live docs are at
https://agenticemail.dev/docs.
