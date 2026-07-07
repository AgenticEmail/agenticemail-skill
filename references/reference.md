# AgenticEmail reference

Base URL: `https://api.agenticemail.dev`. Authenticate every request with
`Authorization: Bearer $AGENTICEMAIL_API_KEY`. Keys are created at
https://app.agenticemail.dev and are scoped per inbox or per account.

## REST

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `/v1/inboxes` | Create an inbox. Body: `{ "username": "assistant" }` (optional; a random one is assigned otherwise). |
| GET | `/v1/inboxes` | List inboxes. |
| POST | `/v1/inboxes/{id}/messages` | Send. Body: `{ "to": ["a@b.com"], "subject": "...", "text": "...", "html": "..." }`. |
| GET | `/v1/inboxes/{id}/messages` | List messages. Query: `limit`, `cursor`. |
| GET | `/v1/inboxes/{id}/messages/{msgId}` | Get one parsed message. |
| GET | `/v1/inboxes/{id}/messages/{msgId}/raw` | Raw MIME (used to fetch ciphertext for E2E). |
| POST | `/v1/inboxes/{id}/messages/{msgId}/reply` | Reply in-thread. Body: `{ "text": "..." }`. |
| POST | `/v1/webhooks` | Register a webhook. Body: `{ "url": "...", "eventTypes": ["message.received"] }`. |
| PUT | `/v1/inboxes/{id}/public-key` | Publish this inbox's public JWKs (E2E). |
| GET | `/v1/public-keys/{address}` | Resolve an address's published public keys. Unauthenticated. |

Every message object includes an `encrypted` boolean so a client knows whether
to decrypt.

## TypeScript SDK (`npm i agenticemail`)

```ts
import { AgenticEmail } from "agenticemail";

const client = new AgenticEmail({ apiKey: process.env.AGENTICEMAIL_API_KEY! });

const inbox = await client.inboxes.create({ username: "assistant" });
await client.messages.send(inbox.id, { to: ["user@example.com"], subject: "Hi", text: "..." });
const { data } = await client.messages.list(inbox.id, { limit: 20 });
const msg = await client.messages.get(inbox.id, data[0].id);
await client.messages.reply(inbox.id, msg.id, { text: "Thanks." });
await client.webhooks.create({ url: "https://you/hook", eventTypes: ["message.received"] });
```

End-to-end encryption: pass an identity and the client encrypts on send and
decrypts on read automatically.

```ts
import { AgenticEmail, generateIdentity } from "agenticemail";

const identity = generateIdentity();
const client = new AgenticEmail({
  apiKey: process.env.AGENTICEMAIL_API_KEY!,
  e2e: { identity, autoPublish: true },
});
```

## Python SDK (`pip install 'agenticemail[e2e]'`)

```python
from agenticemail import AgenticEmail

client = AgenticEmail(api_key=os.environ["AGENTICEMAIL_API_KEY"])
inbox = client.inboxes.create(username="assistant")
client.messages.send(inbox.id, to=["user@example.com"], subject="Hi", text="...")
for m in client.messages.list(inbox.id, limit=20).data:
    print(m.subject)
```

## MCP server (hosted)

Endpoint: `https://api.agenticemail.dev/mcp`. Tools exposed: `create_inbox`,
`list_inboxes`, `list_threads`, `get_thread`, `list_messages`, `get_message`,
`send_message`, `reply_to_message`.

HTTP-native client config (Claude, Cursor, and other MCP clients that speak
streamable HTTP):

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

For a stdio-only client, bridge it:

```bash
npx mcp-remote https://api.agenticemail.dev/mcp \
  --header "Authorization: Bearer $AGENTICEMAIL_API_KEY"
```

## CLI (`npm i -g agenticemail-cli`)

```bash
agenticemail inboxes create
agenticemail messages send <inbox> --to a@b.com --subject "Hi" --text "..."
agenticemail messages list <inbox> --limit 20
agenticemail messages get <inbox> <msgId>
agenticemail wait-for-message <inbox> --timeout 300
agenticemail webhooks create --url https://you/hook --event-types message.received

# end-to-end encryption for this inbox
agenticemail keys generate <inbox>
agenticemail keys publish <inbox>
```

Keys live under `~/.agenticemail/keys` (override with `AGENTICEMAIL_KEY_DIR`) at
`0600`. Once published, `messages send` and `messages get` encrypt and decrypt
automatically.

## What encryption does and does not cover

- It is opt-in and works only between two AgenticEmail inboxes that have each
  published a key. Plaintext is the default.
- The private key is generated and stored client-side and never reaches the
  server, so the platform and its cloud provider cannot read encrypted content.
- A send to an external address (Gmail, a corporate server) falls back to
  plaintext, because there is no published key to encrypt to.
- Routing metadata (From, To, timing, size) stays visible, as it must for SMTP
  to deliver.
- Lose the private key and old encrypted mail is unrecoverable. That is the
  cost of the server never holding it.

Full docs: https://agenticemail.dev/docs
