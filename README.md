
# STOS V2.3.2
**SWAN Telegram Operating System**

One owner. One bot. Buttons replace typing. Every feature must save the owner time.

---

## Architecture

```
INTERNAL MODULES (Layer 1) — Generate intent only. Never mutate state.
  ├── Content Engine
  ├── Button Engine
  ├── Automation Engine
  ├── Community Engine
  ├── Customer Service Engine
  └── Delivery Engine

RUNTIME SERVICES (Layer 2) — Validate intent. Mutate state. Commit reality.
  └── 13-step pipeline:
      1. Event Ingestion
      2. Identity Resolution
      3. Permission Validation
      4. Route Resolution
      5. Execution Planning
      6. FSM Processing
      7. KV Atomic Commit ← all 5 outputs committed in one transaction
      8. Outbox Management
      9. Queue Processing
      10. Delivery Coordination
      11. Audit Recording
      12. Idempotency Control
      13. Lock Management

EXTERNAL TOOLS (Layer 3) — Execute effects. Never mutate STOS state.
  ├── Telegram Bot API
  ├── Deno Runtime / Deno KV
  └── HTTPS Webhook
```

---

## Deployment

### 1. Create a Telegram bot
1. Message [@BotFather](https://t.me/BotFather) → `/newbot`
2. Copy the token

### 2. Get your Telegram user ID
1. Message [@userinfobot](https://t.me/userinfobot)
2. Copy your numeric ID

### 3. Deploy to Deno Deploy
1. Push this repo to GitHub
2. Go to [dash.deno.com](https://dash.deno.com) → New Project → Import from GitHub
3. Set entry point: `main.ts`
4. Set environment variables (from `.env.example`):
   - `BOT_TOKEN`
   - `WEBHOOK_SECRET` (generate: `openssl rand -hex 32`)
   - `OWNER_ID`
   - `WEBHOOK_URL` (your Deno Deploy URL, e.g. `https://stos.deno.dev`)

### 4. Register the webhook
```bash
BOT_TOKEN=... WEBHOOK_SECRET=... WEBHOOK_URL=https://stos.deno.dev \
  deno run --allow-env --allow-net scripts/setup-webhook.ts
```

### 5. Message your bot `/start`
The owner control panel appears.

---

## File Structure

```
stos/
├── main.ts ← Server entry point, webhook handler
├── deno.json ← Project config + task runner
├── .env.example ← Required environment variables
├── scripts/
│ └── setup-webhook.ts ← One-time webhook registration
└── src/
    ├── types/
    │ └── index.ts ← All types and interfaces
    ├── engines/ ← Layer 1: Internal Modules
    │ ├── button-engine.ts ← All inline keyboard rendering
    │ ├── content-engine.ts ← Post lifecycle management
    │ ├── automation-engine.ts ← Broadcasts, reminders, scheduler
    │ ├── community-engine.ts ← Member ingestion, join requests
    │ ├── customer-service-engine.ts ← FAQ, guides, tickets
    │ └── delivery-engine.ts ← Outbox entry builders
    ├── runtime/ ← Layer 2: Runtime Services
    │ ├── pipeline.ts ← 13-step processing pipeline
    │ ├── router.ts ← Step 4: Route resolution
    │ ├── kv.ts ← Deno KV access + atomic commit
    │ ├── delivery.ts ← Step 10: Delivery coordination
    │ └── queue-worker.ts ← Background job processor
    └── utils/
        └── telegram.ts ← Layer 3: Telegram API wrapper
```

---

## Invariants (never violate these)

| # | Rule |
|---|------|
| INV-01 | All mutations commit in one atomic transaction |
| INV-02 | Internal Modules produce intent only. Never side effects |
| INV-03 | Only Runtime Services may mutate persistent state |
| INV-04 | Outbox entries written inside the atomic commit |
| INV-05 | Idempotency check occurs before execution begins |
| INV-06 | Lock acquired before FSM processing begins |
| INV-07 | Audit entries are permanent and immutable |
| INV-08 | STOS has exactly one owner. Always singular |
| INV-09 | Webhook always returns 200 to Telegram |
| INV-10 | Secret token validated on every incoming request |
| INV-11 | Modules do not share state or call each other |
| INV-12 | Pipeline steps execute in fixed sequential order |
| INV-13 | Telegram is an output surface only. Never drives FSM directly |

---

## Security

- Webhook requests without correct secret token → `403` (no processing)
- `BOT_TOKEN` and `WEBHOOK_SECRET` live only in Deno Deploy env vars
- Never in source code. Never in logs. Never in responses.
- Three roles: `OWNER` / `MEMBER` / `GUEST`
- KV namespaces isolated by key prefix, not if-checks

---

*If code diverges from the specification, the code is wrong.*

