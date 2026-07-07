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

### MCP client config

HTTP-native clients (Claude, Cursor, and others that speak streamable HTTP):

```json
{
  "mcpServers": {
    "agenticemail": {
      "type": "http",
      "url": "https://api.agenticemail.dev/mcp",
      "headers": { "Authorization": "Bearer ${AGENTICEMAIL_API_KEY}" }
    }
  }
}
```

Stdio-only client, bridge it:

```bash
npx mcp-remote https://api.agenticemail.dev/mcp \
  --header "Authorization: Bearer $AGENTICEMAIL_API_KEY"
```

## Core flow: create, send, receive, reply

CLI:

```bash
INBOX=$(agenticemail inboxes create --username assistant | jq -r .id)
agenticemail messages send "$INBOX" --to user@example.com \
  --subject "Hello" --text "Sent by an agent."
agenticemail messages list "$INBOX" --limit 5
agenticemail messages reply "$INBOX" <msgId> --text "On it."
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
const msg = await client.messages.get(inbox.id, data[0].id);
await client.messages.reply(inbox.id, msg.id, { text: "Thanks." });
```

Python SDK:

```python
from agenticemail import AgenticEmail
client = AgenticEmail(api_key=os.environ["AGENTICEMAIL_API_KEY"])
inbox = client.inboxes.create(username="assistant")
client.messages.send(inbox.id, to=["user@example.com"], subject="Hello", text="...")
for m in client.messages.list(inbox.id, limit=5).data:
    print(m.subject)
```

## REST (if you are not using an SDK)

Base URL `https://api.agenticemail.dev`, header `Authorization: Bearer $AGENTICEMAIL_API_KEY`.

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/v1/inboxes` | Create an inbox. Body `{ "username": "assistant" }` (optional). |
| GET | `/v1/inboxes` | List inboxes. |
| POST | `/v1/inboxes/{id}/messages` | Send. Body `{ "to": ["a@b.com"], "subject": "...", "text": "..." }`. |
| GET | `/v1/inboxes/{id}/messages` | List. Query `limit`, `cursor`. |
| GET | `/v1/inboxes/{id}/messages/{msgId}` | Get one parsed message. |
| POST | `/v1/inboxes/{id}/messages/{msgId}/reply` | Reply in-thread. |
| POST | `/v1/webhooks` | Register a webhook. Body `{ "url": "...", "eventTypes": ["message.received"] }`. |

Every message object has an `encrypted` boolean so a client knows whether to decrypt.

## Receiving inbound mail

Register a webhook so new mail reaches the agent the moment it arrives:

```bash
agenticemail webhooks create --url https://your-app/hook \
  --event-types message.received
```

Each event carries the parsed sender, subject, text, HTML, and attachments. A
WebSocket stream is available for the same events if the agent is already
running (`agenticemail events tail`).

## End-to-end encryption (optional)

For agent-to-agent mail, AgenticEmail offers opt-in end-to-end encryption: the
private key is generated and kept on the client (SDK or CLI), never on the
server, so the platform cannot read the content. Enable it in the CLI with
`agenticemail keys generate <inbox>` then `agenticemail keys publish <inbox>`,
after which `messages send` and `messages get` encrypt and decrypt
automatically. In the SDK, pass an `e2e` identity when constructing the client:

```ts
import { AgenticEmail, generateIdentity } from "agenticemail";
const client = new AgenticEmail({
  apiKey: process.env.AGENTICEMAIL_API_KEY!,
  e2e: { identity: generateIdentity(), autoPublish: true },
});
```

What it does and does not cover, stated honestly:

- It is opt-in and works only between two AgenticEmail inboxes that have each
  published a key. Plaintext is the default.
- The private key is client-side only and never reaches the server, so neither
  the platform nor its cloud provider can read encrypted content.
- A send to an external address (Gmail, a corporate server) falls back to
  plaintext, because there is no published key to encrypt to.
- Routing metadata (From, To, timing, size) stays visible, as SMTP requires.
- Lose the private key and old encrypted mail is unrecoverable. That is the
  cost of the server never holding it.

## Docs

Full documentation: https://agenticemail.dev/docs
